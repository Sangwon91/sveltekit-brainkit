# Supabase Authentication Flow & Strategy

## Authentication Methods

Supabase supports various authentication methods:
1.  **Email/Password**: Traditional combination.
2.  **OAuth Providers**: Google, GitHub, Apple, etc.
3.  **Magic Link**: Passwordless email links.
4.  **Phone Authentication**: SMS-based.
5.  **Anonymous Sign-in**: For temporary guest sessions.

## Authentication Flow & Token Management

### 1. The Process

```
User Login Request
  ↓
Supabase Auth API Call
  ↓
Success -> Issue JWT
  ↓
Create Access Token + Refresh Token
  ↓
Store in Cookies (httpOnly, secure)
```

### 2. JWT Structure

-   **Access Token**: Short-lived (default 1h), used for API requests.
-   **Refresh Token**: Long-lived (default 30d), used to renew Access Token.
-   **Payload**:
    -   `sub`: User ID (UUID)
    -   `email`: User Email
    -   `role`: Postgres role (`authenticated` or `anon`)
    -   `aud`: Audience (usually `authenticated`)

### 3. Session Storage

-   **Browser**: HTTP-only cookies (prevents XSS).
-   **Key Names**: `sb-{project-ref}-auth-token`, etc.
-   **Auto-Refresh**: Handled by Supabase client before access token expiry.

### 4. Propagation Path

```
Browser Cookie
  ↓
Next.js Proxy (Reads Cookie)
  ↓
Supabase Client (Extracts Token)
  ↓
API Request (Authorization Header)
  ↓
FastAPI (JWT Verification)
```

## Defense in Depth Strategy

### Layer 1: Next.js Proxy (First Line of Defense)

-   **Responsibility**: Auth check, blocking unauthenticated users.
-   **Location**: `proxy.ts` (Next.js 16+ replacement for middleware).
-   **Action**: Redirect to `/login` if authentication fails.

**Implementation Example:**

```typescript
// proxy.ts
import { type NextRequest, NextResponse } from "next/server";
import { createMiddlewareClient, updateSession } from "@/lib/supabase/middleware";

export async function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const publicPaths = ["/login", "/auth/callback", "/auth/auth-code-error"];
  
  if (publicPaths.some((path) => pathname.startsWith(path))) {
    return await updateSession(request);
  }

  const { supabase, getResponse } = createMiddlewareClient(request);
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("redirect", pathname);
    return NextResponse.redirect(loginUrl);
  }

  return getResponse();
}
```

**Alternative: Using `getClaims()` for JWT-only verification**

If you only need to verify the JWT token without fetching full user data from the Auth server, use `getClaims()` for better performance:

```typescript
// proxy.ts (Alternative approach)
import { type NextRequest, NextResponse } from "next/server";
import { createMiddlewareClient, updateSession } from "@/lib/supabase/middleware";

export async function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const publicPaths = ["/login", "/auth/callback", "/auth/auth-code-error"];
  
  if (publicPaths.some((path) => pathname.startsWith(path))) {
    return await updateSession(request);
  }

  const { supabase, getResponse } = createMiddlewareClient(request);
  
  // Extract JWT from session
  const { data: { session } } = await supabase.auth.getSession();
  if (!session?.access_token) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("redirect", pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Use getClaims() for fast JWT verification via JWKS
  const { data: claims, error } = await supabase.auth.getClaims({
    access_token: session.access_token
  });

  if (error || !claims) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("redirect", pathname);
    return NextResponse.redirect(loginUrl);
  }

  return getResponse();
}
```

**When to use `getUser()` vs `getClaims()`:**

- **Use `getUser()`** when:
  - You need full user object with metadata
  - You need to verify session is still valid on Auth server
  - You're in Next.js Server Components with cookie-based sessions
  
- **Use `getClaims()`** when:
  - You only need JWT claims (user ID, email, role)
  - Performance is critical (no Auth server round-trip)
  - You have the JWT token directly (e.g., from Authorization header)
  - You're in FastAPI or API routes with Bearer tokens

### Layer 2: FastAPI Middleware (Second Line of Defense)

-   **Responsibility**: JWT validation for API requests.
-   **Validator**: Supabase JWKS.
-   **Action**: Return 401 Unauthorized if invalid.
