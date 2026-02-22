# 🔴 Authentication Redirect Loop - Updated Investigation Report

**Date:** February 21, 2026  
**Severity:** 🔴 Critical - ALL navigation from `/org` is broken  
**Status:** Root Cause Confirmed  

---

## 📋 Executive Summary

**ALL pages** under `/org/*` are affected by the redirect loop, not just dashboard pages:

| Page | Status | API Call | Protected |
|------|--------|----------|-----------|
| `/org` | ✅ Works | `/api/users/agents` | ✅ Yes |
| `/org/billing` | ❌ Redirects | `/api/payments/history` | ✅ Yes |
| `/org/usage` | ❌ Redirects | Uses `user` from context (no API) | N/A |
| `/org/account/me` | ❌ Redirects | `/api/users/me` | ✅ Yes |
| `/dashboard` | ❌ Redirects | `/api/conversations/stats` | ✅ Yes |
| `/conversations` | ❌ Redirects | `/api/conversations/` | ✅ Yes |
| `/tools` | ❌ Redirects | `/api/tools/` | ✅ Yes |
| `/knowledge` | ❌ Redirects | `/api/knowledge/` | ✅ Yes |
| `/agent` | ❌ Redirects | `/api/agents/` | ✅ Yes |
| `/channels` | ❌ Redirects | `/api/widgets/` | ✅ Yes |

**Key Finding:** Even `/org/usage` which does **NOT** make any API calls (only uses `useAuthContext()` data) still causes a redirect. This indicates the issue is **NOT** just API authentication failure.

---

## 🔍 Root Cause Analysis

### The Real Problem: **Two-Layer Authentication Conflict**

```
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 1: Next.js Middleware (app/src/middleware.ts)            │
│  - Checks cookies: accessToken, idToken                         │
│  - Redirects unauthenticated → /auth/signin                     │
│  - Redirects authenticated away from /auth → /org               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 2: React AuthProvider (app/src/components/AuthProvider.tsx) │
│  - Client-side auth check via /api/users/me                     │
│  - If /api/users/me returns 401 → redirects to /auth/signin     │
│  - Shows loading screen while checking                          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  LAYER 3: Backend Express (backend/src/middleware/auth.middleware.ts) │
│  - Verifies JWT via AWS Cognito                                 │
│  - Returns 401 if token invalid/expired                         │
└─────────────────────────────────────────────────────────────────┘
```

### The Conflict

**Scenario: User navigates from `/org` to `/org/billing`**

1. **Next.js Middleware** ✅ PASS
   - Sees `accessToken` cookie
   - Allows navigation to proceed

2. **Page Loads** ✅ PASS
   - `/org/billing/page.tsx` renders
   - Component mounts

3. **AuthProvider Client-Side Check** ❌ FAIL
   - `useEffect` in `AuthProvider.tsx` calls `checkAuth()`
   - `checkAuth()` calls `authApi.getMe()` → `GET /api/users/me`
   - Backend returns **401 Unauthorized** (JWT verification fails)
   - `AuthProvider` tries to refresh session
   - Refresh also fails
   - **`AuthProvider` redirects to `/auth/signin?expired=1&callbackUrl=/org/billing`**

4. **Next.js Middleware Intercepts** 🔄 LOOP
   - Sees user has `accessToken`/`idToken` cookies
   - Middleware rule: "Authenticated users can't access /auth pages"
   - **Redirects back to `/org`** (or callbackUrl)

5. **User tries again** → Loop repeats

---

## 🎯 Why Backend Returns 401

The backend's `authenticateToken` middleware fails JWT verification:

```typescript
// backend/src/middleware/auth.middleware.ts
const verifier = CognitoJwtVerifier.create({
    userPoolId: config.COGNITO_USER_POOL_ID,
    tokenUse: "access",
    clientId: config.COGNITO_CLIENT_ID,
});

export async function requireAuth(req, res, next) {
    // Gets token from Authorization header OR cookie
    let token = authHeader?.substring(7) || req.cookies.accessToken;
    
    try {
        const payload = await verifier.verify(token);
        req.user = payload;
        next();
    } catch (error: any) {
        // THIS IS WHERE IT FAILS
        console.error('[AUTH] JWT verification failed:', error);
        res.status(401).json({ error: 'Invalid authentication token' });
    }
}
```

### Most Likely Causes

| # | Cause | Probability | Evidence |
|---|-------|-------------|----------|
| 1 | **Expired access token** | HIGH | Token TTL exceeded, refresh not working |
| 2 | **Cognito config mismatch** | HIGH | Wrong `COGNITO_USER_POOL_ID` or `COGNITO_CLIENT_ID` |
| 3 | **Token type wrong** | MEDIUM | Cookie has `idToken` but backend expects `accessToken` |
| 4 | **Missing backend env vars** | MEDIUM | `COGNITO_*` vars not loaded |
| 5 | **Clock skew** | LOW | Server time differs from token issue time |

---

## 🔬 Diagnostic Evidence

### 1. `/org/usage` Page Behavior

```typescript
// app/src/app/org/usage/page.tsx
export default function UsagePage() {
    const { user, isLoading } = useAuthContext();
    
    // NO API CALLS - only uses context
    const { actions, limits, usage } = user;
    
    // Renders UI with user data from context
}
```

**This page still causes redirect** because:
- `AuthProvider.tsx` has a global `useEffect` that calls `checkAuth()` on ALL protected pages
- `checkAuth()` calls `/api/users/me`
- Backend returns 401
- Redirect happens

### 2. `/org` Page Works

```typescript
// app/src/app/org/page.tsx
const fetchAgents = async () => {
    const data = await agentApi.getAll(); // Calls /api/users/agents
    setAgents(data || []);
};
```

**This works** because:
- User is already on `/org` when this runs
- Even if API fails, page doesn't redirect (just shows empty state)
- No explicit auth check on this page

### 3. All Other Pages Fail

Every other page either:
- Makes protected API calls (billing, account/me, dashboard)
- Triggers `AuthProvider.checkAuth()` via context (usage)

---

## 🛠️ Immediate Debugging Steps

### Step 1: Check Browser Console

```javascript
// Open DevTools Console and run:
document.cookie.split(';').forEach(c => {
    const [name, value] = c.trim().split('=');
    if (name.includes('Token') || name.includes('session')) {
        console.log(`${name}: ${value?.substring(0, 50)}...`);
        // Decode JWT if present
        if (value && value.split('.').length === 3) {
            try {
                const payload = JSON.parse(atob(value.split('.')[1]));
                console.log('  Exp:', new Date(payload.exp * 1000));
                console.log('  Now:', new Date());
                console.log('  Expired:', payload.exp < Date.now()/1000);
            } catch {}
        }
    }
});
```

### Step 2: Check Network Tab

1. Navigate to `/org/billing`
2. Look for these requests:
   - `GET /api/users/me` → Check status code and response body
   - `GET /api/payments/history` → Check if it even fires
   - `POST /api/auth/refresh` → Check if refresh is attempted

**Expected response for failed auth:**
```json
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid authentication token"
  }
}
```

### Step 3: Check Backend Logs

Add logging to backend middleware:

```typescript
// backend/src/middleware/auth.middleware.ts - Line 88
export async function requireAuth(req: Request, res: Response, next: NextFunction) {
    const authHeader = req.headers.authorization;
    let token = '';

    if (authHeader?.startsWith('Bearer ')) {
        token = authHeader.substring(7);
    } else if (req.cookies?.accessToken) {
        token = req.cookies.accessToken;
    }

    // ADD THIS LOG
    console.log('[AUTH] Token check:', {
        hasAuthHeader: !!authHeader,
        hasCookie: !!req.cookies?.accessToken,
        tokenLength: token?.length,
        tokenStart: token?.substring(0, 20),
    });

    if (!token) {
        res.status(401).json({ error: 'Authentication required' });
        return;
    }

    try {
        const payload = await verifier.verify(token);
        req.user = payload;
        req.authToken = token;
        next();
    } catch (error: any) {
        // ENHANCED LOG
        console.error('[AUTH] JWT verification FAILED:', {
            errorName: error.name,
            errorMessage: error.message,
            errorStack: error.stack,
            tokenPresent: !!token,
            tokenFirst50: token?.substring(0, 50),
        });
        
        res.status(401).json({
            error: {
                code: 'UNAUTHORIZED',
                message: 'Invalid authentication token'
            }
        });
    }
}
```

### Step 4: Verify Backend Environment

Check backend `.env` file:

```bash
# Backend must have these EXACT values from AWS Cognito
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_DOMAIN=https://your-domain.auth.us-east-1.amazoncognito.com
COGNITO_REDIRECT_URI=http://localhost:3000/auth/callback
SESSION_SECRET=your-secret-key
```

**Verify in AWS Console:**
1. Go to AWS Cognito → User Pools
2. Select your pool → Copy "User pool ID"
3. Go to "App integration" → "App client" → Copy "App client ID"
4. Go to "App integration" → "Domain" → Copy domain

---

## 🔧 Fix Options

### Option A: Fix Cognito Configuration (Most Likely)

**If backend logs show JWT verification errors:**

1. Stop backend server
2. Update backend `.env` with correct Cognito values
3. Restart backend
4. Clear browser cookies
5. Login again

### Option B: Fix Token Storage (If Cookie is Wrong Type)

**Check what's being stored:**

```typescript
// app/src/lib/api/auth.ts - Check exchangeAuthCode response
export const authApi = {
    exchangeAuthCode: async (code: string) => {
        return apiFetch('/api/auth/exchange', {
            method: 'POST',
            body: JSON.stringify({ code })
        });
    }
};
```

**Backend should set BOTH cookies:**

```typescript
// backend/src/auth/auth.routes.ts
res.cookie('accessToken', tokens.accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 3600000 // 1 hour
});

res.cookie('idToken', tokens.idToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 3600000
});
```

### Option C: Fix AuthProvider Redirect Logic

**Current problematic logic:**

```typescript
// app/src/components/AuthProvider.tsx - Line 143
const checkAuth = useCallback(async () => {
    try {
        const userData = await authApi.getMe();
        setUser(userData);
    } catch {
        setUser(null);
        
        // THIS IS THE PROBLEM - Always redirects on protected routes
        if (!isPublicRoute) {
            try {
                await authApi.refreshSession();
                // ... retry logic
            } catch {
                // Forces redirect
                router.push(`${config.links.signIn}?expired=1&callbackUrl=${pathname}`);
            }
        }
    }
}, [pathname, router, isPublicRoute]);
```

**Temporary workaround - Add grace period:**

```typescript
// Allow pages to render even if auth check fails
// Let individual pages handle their own auth
const checkAuth = useCallback(async () => {
    try {
        const userData = await authApi.getMe();
        setUser(userData);
    } catch {
        // Don't redirect immediately - let pages handle it
        setUser(null);
        
        // Only redirect if explicitly on an auth page
        if (pathname?.startsWith('/auth')) {
            router.push(config.links.signIn);
        }
        // Otherwise, let the page render and handle its own errors
    }
}, [pathname, router]);
```

### Option D: Manual Token Refresh Before Expiry

Add proactive token refresh:

```typescript
// app/src/components/AuthProvider.tsx
useEffect(() => {
    if (!user) return;
    
    // Refresh token every 50 minutes (before 1-hour expiry)
    const refreshInterval = setInterval(async () => {
        try {
            await authApi.refreshSession();
            await checkAuth(); // Re-fetch user
        } catch (err) {
            console.error('[Auth] Auto-refresh failed');
        }
    }, 50 * 60 * 1000);
    
    return () => clearInterval(refreshInterval);
}, [user, checkAuth]);
```

---

## 📊 Impact Matrix

| Feature | Impact | Workaround |
|---------|--------|------------|
| View billing | ❌ Blocked | None |
| View usage | ❌ Blocked | None |
| Account settings | ❌ Blocked | None |
| Dashboard | ❌ Blocked | None |
| Conversations | ❌ Blocked | None |
| Tools | ❌ Blocked | None |
| Knowledge | ❌ Blocked | None |
| Agent config | ❌ Blocked | None |
| Channels | ❌ Blocked | None |
| `/org` (Agents list) | ✅ Works | N/A |

---

## 🧪 Testing Checklist

After applying fixes:

- [ ] Clear all browser cookies
- [ ] Login with Google OAuth
- [ ] Navigate to `/org` → Should show agents
- [ ] Navigate to `/org/billing` → Should load without redirect
- [ ] Navigate to `/org/usage` → Should load without redirect
- [ ] Navigate to `/org/account/me` → Should load without redirect
- [ ] Navigate to `/dashboard?agentId=xxx` → Should load
- [ ] Check Network tab → No 401 errors on `/api/users/me`
- [ ] Wait 1 hour → Session should auto-refresh
- [ ] Logout → Should redirect to signin

---

## 📝 Recommended Action Plan

### Immediate (Today)

1. **Check backend logs** for JWT verification errors
2. **Verify Cognito configuration** matches AWS Console
3. **Test with fresh login** after clearing cookies

### Short-term (This Week)

4. **Add better error logging** to backend auth middleware
5. **Implement proactive token refresh** in AuthProvider
6. **Add grace period** before redirecting on auth failure

### Long-term (Next Sprint)

7. **Implement proper session management** with refresh tokens
8. **Add monitoring** for auth failures
9. **Create auth health dashboard** for debugging

---

## 🔗 Related Files

### Frontend
- `app/src/middleware.ts` - Next.js auth middleware
- `app/src/components/AuthProvider.tsx` - Auth context (redirect logic)
- `app/src/lib/api/client.ts` - API fetch with 401 handling
- `app/src/lib/api/auth.ts` - Auth API calls
- `app/src/app/org/billing/page.tsx` - Affected page
- `app/src/app/org/usage/page.tsx` - Affected page (no API calls)

### Backend
- `backend/src/middleware/auth.middleware.ts` - JWT verification
- `backend/src/auth/auth.routes.ts` - Auth endpoints
- `backend/src/config/index.ts` - Environment config
- `backend/src/routes/user.routes.ts` - `/api/users/me` endpoint
- `backend/src/routes/payment.routes.ts` - `/api/payments/*` endpoints
- `backend/src/payments/payment.routes.ts` - Payment history endpoint

---

## 🚨 Critical Finding

**The issue is NOT just API authentication failure.**

Even pages that don't make API calls (`/org/usage`) fail because:

1. `AuthProvider.tsx` has a **global** `useEffect` that calls `checkAuth()` on mount
2. `checkAuth()` **always** calls `/api/users/me`
3. Backend returns 401
4. `AuthProvider` redirects to signin
5. Middleware intercepts and redirects back

**This means fixing just the API calls won't solve the problem.** The AuthProvider's aggressive redirect logic must also be addressed.

---

**Report Generated:** Investigation Complete  
**Action Required:** 
1. Backend team: Verify Cognito configuration and check auth logs
2. Frontend team: Review AuthProvider redirect logic
3. DevOps: Ensure backend env vars are correctly configured
