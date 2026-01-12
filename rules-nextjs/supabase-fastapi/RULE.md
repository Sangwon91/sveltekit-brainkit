---
alwaysApply: false
---

# Supabase & FastAPI Integration Rules

## Philosophy
**Defense in Depth & Pragmatic Boundaries.**
We maintain strict separation of concerns: Next.js handles user sessions and standard data access (via RLS), while FastAPI handles complex logic. Security is enforced at both layers: Next.js Proxy checks sessions, and FastAPI validates JWTs.

## Core Knowledge (Cheat Sheet)

-   **Auth Source**: Supabase Auth (JWT) is the single source of truth.
-   **Session Default**: Use `httpOnly` cookies. Next.js Proxy (`proxy.ts`) manages session refresh.
-   **Data Access**: Prefer querying Supabase directly from Next.js (`queries.ts`). Use FastAPI only for complex logic.
-   **Service Role**: Only usage in FastAPI is allowed. **NEVER** in Client. Validating user permissions manually is **MANDATORY** when using Service Role.

### Tech Stack
-   **Auth**: `@supabase/ssr` (Next.js), `supabase.auth.get_claims()` (FastAPI)
-   **Database**: Postgres (via Supabase Client)
-   **Middleware**: `createMiddlewareClient` (Next.js), Custom Auth Middleware (FastAPI)

## Implementation Checklist

1.  **Next.js Proxy**: Ensure `proxy.ts` handles auth redirects and session refresh.
2.  **FastAPI Auth**: Add Middleware to validate `Authorization: Bearer <token>`.
3.  **RLS**: Enable RLS on all tables. Define policies for `authenticated` role.
4.  **Secret Key**: Audit FastAPI code. Ensure every Secret Key query filters by `user_id` (legacy: Service Role).
5.  **Environment**: Check `SUPABASE_SECRET_KEY` is set ONLY on server environments (legacy: `SUPABASE_SERVICE_ROLE_KEY`).

## References

-   See [Authentication Flow & Tokens](references/auth_flow.md) for detailed token lifecycle.
-   See [Access Control Patterns](references/access_control.md) for RBAC and permission checks.
-   See [Data Access Patterns](references/data_access.md) for deciding between Next.js Direct vs FastAPI.
-   See [Security Best Practices](references/security_best_practices.md) for critical security rules.
-   See [FastAPI Integration](references/fastapi_integration.md) for Python-specific patterns.