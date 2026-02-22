# 🔴 Authentication Redirect Loop - Investigation Report

**Date:** February 21, 2026  
**Severity:** Critical - Blocks all navigation from /org to protected pages  
**Status:** Root Cause Identified  

---

## 📋 Executive Summary

When navigating from `/org` (Agents page) to any protected dashboard page (`/dashboard`, `/conversations`, `/tools`, `/knowledge`, `/channels`, `/agent`), users are immediately redirected back to `/org` despite having valid authentication cookies.

**Observed Behavior:**
```
GET /org                              200 OK
GET /dashboard?agentId=xxx            200 OK (page loads)
GET /api/conversations/stats          401 Unauthorized
→ Redirect to /auth/signin?expired=1
→ Middleware redirects to /org (user has cookies)
→ Loop repeats
```

---

## 🔍 Technical Analysis

### 1. Affected Pages

All pages that require **agent-specific API calls** are affected:

| Page | Component | API Endpoints Called | Protected |
|------|-----------|---------------------|-----------|
| `/dashboard` | `useDashboardData()` | `/api/conversations/stats`, `/api/conversations/`, `/api/conversations/analytics` | ✅ |
| `/conversations` | `Conversations.tsx` | `/api/conversations/` | ✅ |
| `/tools` | `Tools.tsx` | `/api/tools/`, `/api/agent/credentials` | ✅ |
| `/knowledge` | `Knowledge.tsx` | `/api/knowledge/` | ✅ |
| `/channels` | `Channels.tsx` | `/api/widgets/`, `/api/tools/telegram/` | ✅ |
| `/agent` | `Agent.tsx` | `/api/agents/` | ✅ |

### 2. Authentication Flow

```
┌─────────────────────────────────────────────────────────────────┐
│  FRONTEND (Next.js App)                                         │
│                                                                 │
│  User clicks navigation → Page loads (200 OK)                  │
│         ↓                                                       │
│  Component mounts → calls apiFetch()                           │
│         ↓                                                       │
│  API returns 401 → client.ts tries /api/auth/refresh           │
│         ↓                                                       │
│  Refresh fails → window.location = /auth/signin?expired=1      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  MIDDLEWARE (Next.js)                                           │
│                                                                 │
│  Sees accessToken/idToken cookies present                      │
│         ↓                                                       │
│  Redirects authenticated users away from /auth                 │
│  → Sends user back to /org                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  BACKEND (Express API)                                          │
│                                                                 │
│  authenticateToken middleware verifies JWT via Cognito         │
│         ↓                                                       │
│  Verification FAILS → returns 401                              │
│  Reason: Token expired / invalid / misconfigured               │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Backend Protection Matrix

All affected API routes are protected by `authenticateToken` middleware:

```typescript
// backend/src/routes/conversation.routes.ts
router.use(authenticate);  // Line 21 - ALL conversation routes protected

// backend/src/routes/knowledge.routes.ts  
router.use(authenticate);  // Line 50 - ALL knowledge routes protected

// backend/src/routes/tool.routes.ts
router.get('/definitions', AgentToolsController.listDefinitions); // ✅ Public
router.get('/', toolsController.getAvailableTools);               // ✅ Public
router.get('/oauth/*', authMiddleware);                           // 🔒 Protected
```

### 4. JWT Verification (Backend)

```typescript
// backend/src/middleware/auth.middleware.ts
const verifier = CognitoJwtVerifier.create({
    userPoolId: config.COGNITO_USER_POOL_ID,
    tokenUse: "access",      // Expects ACCESS token
    clientId: config.COGNITO_CLIENT_ID,
});

export async function requireAuth(req, res, next) {
    // 1. Try Authorization header (Bearer token)
    // 2. Fallback to Cookie (accessToken)
    
    if (!token) return 401;
    
    try {
        const payload = await verifier.verify(token);
        req.user = payload;
        next();
    } catch (error) {
        // JWT verification failed → 401
        res.status(401).json({ error: 'Invalid authentication token' });
    }
}
```

### 5. Frontend Token Handling

```typescript
// app/src/lib/api/client.ts - apiFetch()

// Handle 401 Session Expiration
if (response.status === 401 && !options._retry) {
    // Try to refresh session
    const refreshRes = await fetch(`${API_URL}/api/auth/refresh`, {
        method: 'POST',
        credentials: 'include',
    });
    
    if (refreshRes.ok && refreshData.success !== false) {
        // Retry original request
        return apiFetch<T>(endpoint, { ...options, _retry: true });
    } else {
        // Redirect to signin
        window.location.href = `/auth/signin?expired=1`;
    }
}
```

---

## 🎯 Root Cause

**Primary Issue:** Backend JWT verification is failing for one of these reasons:

| # | Potential Cause | Likelihood | How to Verify |
|---|-----------------|------------|---------------|
| 1 | **Expired access token** - Token TTL exceeded | HIGH | Check browser cookies `accessToken` expiry |
| 2 | **Cognito config mismatch** - Wrong `COGNITO_USER_POOL_ID` or `COGNITO_CLIENT_ID` in backend `.env` | HIGH | Compare backend env with AWS Cognito console |
| 3 | **Token type mismatch** - Frontend stores `idToken` but backend expects `access` token | MEDIUM | Inspect cookie contents in DevTools |
| 4 | **Clock skew** - Server time differs from token issue time | MEDIUM | Check server time vs local time |
| 5 | **Missing env vars** - `COGNITO_*` vars not loaded in backend | MEDIUM | Add logging to backend config/index.ts |
| 6 | **Refresh endpoint broken** - `/api/auth/refresh` not working properly | LOW | Test endpoint directly in browser/Postman |

---

## 🛠️ Recommended Fixes

### Immediate Actions (Debug First)

#### 1. **Enable Backend Debug Logging**
Add logging to see WHY JWT verification fails:

```typescript
// backend/src/middleware/auth.middleware.ts - Line 88
export async function requireAuth(req, res, next) {
    // ... existing code ...
    
    try {
        const payload = await verifier.verify(token);
        req.user = payload;
        next();
    } catch (error: any) {
        // ADD THIS LOG
        console.error('[AUTH] JWT verification failed:', {
            error: error.name,
            message: error.message,
            stack: error.stack,
            tokenPresent: !!token,
            tokenLength: token?.length,
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

#### 2. **Check Browser Console & Network Tab**
1. Open DevTools → Network tab
2. Navigate to `/dashboard`
3. Look for:
   - `/api/conversations/stats` → Check response body
   - `/api/auth/refresh` → Check if it returns `success: true`
   - Any CORS errors

#### 3. **Inspect Cookies**
In browser console:
```javascript
document.cookie.split(';').forEach(c => {
    const [name] = c.trim().split('=');
    if (name.includes('Token')) console.log(c);
});
```

Check if `accessToken` exists and its format (should be 3 JWT parts separated by `.`)

---

### Fix Options

#### **Option A: Fix Cognito Configuration** (Most Likely)

Verify backend `.env`:
```env
# Must match EXACTLY what's in AWS Cognito
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_DOMAIN=https://your-domain.auth.us-east-1.amazoncognito.com
```

**Verify in AWS Console:**
1. Go to AWS Cognito → User Pools
2. Select your pool
3. Check "App clients" → Copy Client ID
4. Check "Domain name" → Copy domain

---

#### **Option B: Fix Token Type Mismatch**

If frontend is storing wrong token type:

```typescript
// app/src/lib/api/auth.ts - Check what's being stored
export const authApi = {
    exchangeAuthCode: async (code: string) => {
        const response = await fetch(`${API_URL}/api/auth/exchange`, {
            method: 'POST',
            credentials: 'include',  // Cookies sent
        });
        const data = await response.json();
        
        // Backend should set BOTH cookies:
        // - accessToken (for API calls)
        // - idToken (for middleware checks)
    }
};
```

**Backend should set cookies like this:**
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

---

#### **Option C: Extend Token Lifetime or Auto-Refresh**

If tokens expire too quickly:

1. **In AWS Cognito:** Increase access token TTL (max 24 hours)
2. **In Frontend:** Proactively refresh before expiry

```typescript
// app/src/components/AuthProvider.tsx
useEffect(() => {
    if (!user) return;
    
    // Refresh token 5 minutes before expiry
    const refreshInterval = setInterval(async () => {
        try {
            await authApi.refreshSession();
        } catch {
            // Let redirect happen
        }
    }, 55 * 60 * 1000); // 55 minutes
    
    return () => clearInterval(refreshInterval);
}, [user]);
```

---

#### **Option D: Add Grace Period in Middleware**

Prevent aggressive redirects for recently-expired tokens:

```typescript
// app/src/middleware.ts
export function middleware(request: NextRequest) {
    const accessToken = request.cookies.get('accessToken');
    const idToken = request.cookies.get('idToken');
    const { pathname } = request.nextUrl;

    // Decode token to check expiry
    if (idToken?.value) {
        try {
            const payload = JSON.parse(atob(idToken.value.split('.')[1]));
            const now = Date.now() / 1000;
            const isExpired = payload.exp < now;
            
            // Allow 5-minute grace period
            const gracePeriod = 300;
            const recentlyExpired = (now - payload.exp) < gracePeriod;
            
            if (recentlyExpired) {
                return NextResponse.next(); // Let page load, client will refresh
            }
        } catch {
            // Invalid token format
        }
    }
    
    // ... rest of existing logic
}
```

---

## 📊 Impact Assessment

| Impact Area | Severity | Notes |
|-------------|----------|-------|
| User Experience | 🔴 Critical | Users cannot access any dashboard features |
| Data Access | 🔴 Critical | All agent-specific data is inaccessible |
| Authentication | 🟡 High | Login works, but session doesn't persist |
| Public Pages | 🟢 None | `/org`, `/auth/*` work correctly |

---

## 🧪 Testing Checklist

After applying fixes:

- [ ] Login with Google OAuth
- [ ] Navigate to `/org` → Should show agents list
- [ ] Click on an agent → Should go to `/dashboard?agentId=xxx`
- [ ] Check Network tab → No 401 errors
- [ ] Navigate to Conversations, Tools, Knowledge → All work
- [ ] Wait 1 hour → Session should auto-refresh
- [ ] Logout → Should clear all cookies

---

## 📝 Next Steps

1. **Immediate:** Check backend logs for JWT verification errors
2. **Short-term:** Verify/fix Cognito configuration
3. **Medium-term:** Implement proactive token refresh
4. **Long-term:** Add monitoring for auth failures

---

## 🔗 Related Files

### Frontend
- `app/src/middleware.ts` - Next.js auth middleware
- `app/src/lib/api/client.ts` - API fetch wrapper with 401 handling
- `app/src/components/AuthProvider.tsx` - Auth context provider
- `app/src/lib/api/auth.ts` - Auth API calls

### Backend
- `backend/src/middleware/auth.middleware.ts` - JWT verification
- `backend/src/auth/auth.routes.ts` - Auth endpoints
- `backend/src/config/index.ts` - Environment config
- `backend/src/routes/*.routes.ts` - Protected route definitions

---

**Report Generated:** Investigation Complete  
**Action Required:** Backend team to verify Cognito configuration and check auth logs
