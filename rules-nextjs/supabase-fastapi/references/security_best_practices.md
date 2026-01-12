# Security Best Practices

## API Key Management

| Key Type | Env Var Name | Scope | Rules |
|----------|--------------|-------|-------|
| **Publishable** | `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Public / Client | Safe for browser. (Legacy: `NEXT_PUBLIC_SUPABASE_ANON_KEY`) |
| **Secret** | `SUPABASE_SECRET_KEY` | Private / Server | **NEVER** expose to client. Bypasses RLS. (Legacy: `SUPABASE_SERVICE_ROLE_KEY`) |

> **참고**: Supabase는 레거시 `anon`과 `service_role` 키를 새로운 `publishable`과 `secret` 키로 전환 중입니다. 신규 프로젝트는 Publishable/Secret 키 사용을 권장합니다.

## Security Rules

### ✅ DO
-   Use `proxy.ts` for auth checks in Next.js 16+.
-   Pass JWT tokens when calling FastAPI.
-   Manually check permissions when using Service Role Key.
-   Enable RLS on **ALL** Supabase tables.
-   Use `httpOnly` secure cookies.

### ❌ DON'T
-   **NEVER** use Secret Key in Client Components (legacy: Service Role Key).
-   **NEVER** use `NEXT_PUBLIC_` prefix for Secret Key.
-   **NEVER** decode JWTs without verification.
-   **NEVER** disable RLS just to make things work.

## Anti-Patterns

### Bad: Secret Key in Client
```typescript
// ❌ CRITICAL SECURITY RISK (legacy: Service Role Key)
const supabase = createClient(url, SECRET_KEY); 
```

### Bad: Unverified JWT Decoding
```python
# ❌ RISK: Forged tokens possible
payload = jwt.decode(token, options={"verify_signature": False})
```

### Good: Verified JWT with get_claims()
```python
# ✅ SECURE: Automatic signature verification via JWKS
from supabase import create_client

supabase = create_client(url, publishable_key)  # legacy: anon_key
response = supabase.auth.get_claims(jwt=token)
if response.error:
    raise HTTPException(401, "Invalid JWT")
claims = response.data  # Automatically verified via JWKS
```

**Why `get_claims()` is preferred:**
- Automatically verifies JWT signature using JWKS endpoint
- Caches JWKS keys for performance optimization
- Faster than `get_user()` as it doesn't require Auth server round-trip
- Built-in error handling for invalid tokens

### Bad: Missing Manual Auth Check
```python
# ❌ RISK: Data leak
# Secret key bypasses RLS, so this returns ALL users' data (legacy: Service role)
data = supabase.table('users').select('*').execute() 
```
