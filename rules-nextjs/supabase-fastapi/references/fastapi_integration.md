# FastAPI Integration

## Auth Middleware Strategy

FastAPI sits behind Next.js (or is called by it). It acts as a second line of defense.

### Dependency Injection Pattern

Always use `Depends` to extract and validate the User ID from the JWT.

```python
# api/auth/dependencies.py
from fastapi import Depends, HTTPException, Request
from supabase import create_client, Client
from functools import lru_cache
import os

# Initialize Supabase client for JWT verification (uses publishable key for JWKS, legacy: anon key)
@lru_cache()
def get_supabase_auth_client() -> Client:
    return create_client(
        os.environ["SUPABASE_URL"],
        os.environ["NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY"]  # legacy: NEXT_PUBLIC_SUPABASE_ANON_KEY
    )

def get_current_user_id(request: Request) -> str:
    auth_header = request.headers.get("Authorization")
    if not auth_header or not auth_header.startswith("Bearer "):
        raise HTTPException(401, "Missing Authorization Header")
    
    token = auth_header.split(" ")[1]
    supabase = get_supabase_auth_client()
    
    # Use get_claims() for fast JWT verification via JWKS
    response = supabase.auth.get_claims(jwt=token)
    if response.error:
        raise HTTPException(401, f"Invalid JWT: {response.error}")
    
    claims = response.data
    return claims.get("sub")
```

## Local Development Docs

To allow access to `/docs` (Swagger UI) in local development without valid auth headers:

```python
# api/auth/authenticator.py
def should_skip_auth(self, request: Any) -> bool:
    from fastapi import Request
    
    docs_paths = ["/docs", "/redoc", "/openapi.json"]
    if request.url.path in docs_paths:
        # Only allow skipping if on localhost AND not in production
        host = request.headers.get("host")
        if is_localhost_request(host) and self.environment != "production":
            return True
            
    return False
```

## Secret Key Client

Create a clean singleton or dependency for the Secret Key client to avoid resource leaks (legacy: Service Role).

```python
# api/services/supabase.py
@lru_cache()
def get_supabase_secret():
    return create_client(
        os.environ["SUPABASE_URL"],
        os.environ["SUPABASE_SECRET_KEY"]  # legacy: SUPABASE_SERVICE_ROLE_KEY
    )
```
