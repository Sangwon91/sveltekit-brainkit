# Supabase Auth Architecture Rules

## 1. Dual Client Strategy (Backend ↔ Frontend)

### Backend: Service Role Client Only
- FastAPI에서는 **Service Role Client만 사용**하여 RLS 우회
- 권한 검증은 FastAPI 레이어에서 직접 관리 (`require_admin`, `require_admin_or_system`)
- 권한 투명성 확보 및 디버깅 용이

### Frontend: User Client with RLS
- SvelteKit에서는 **User Client** 사용 (JWT 기반)
- RLS 정책이 DB 레벨에서 적용됨
- `@supabase/ssr` 사용하여 SSR 호환

```
┌──────────────────┐      ┌──────────────────┐
│   Frontend       │      │   Backend        │
│   (SvelteKit)    │      │   (FastAPI)      │
├──────────────────┤      ├──────────────────┤
│ User Client      │      │ Service Client   │
│ JWT Token        │      │ Service Role Key │
│ RLS Enforced     │      │ RLS Bypassed     │
└────────┬─────────┘      └────────┬─────────┘
         │                         │
         ▼                         ▼
┌─────────────────────────────────────────────┐
│         Supabase (PostgreSQL + RLS)         │
│   RLS Enforced for User Client Only         │
│   (Service Role Client bypasses all RLS)    │
└─────────────────────────────────────────────┘
```

## 2. JWT Signing Keys (Asymmetric)

### 현재 설정: 비대칭 키 (RSA/Ed25519)
- JWKS Endpoint: `https://<project>.supabase.co/auth/v1/.well-known/jwks.json`
- Public Key로 로컬 JWT 검증 가능

### getClaims() 사용 권장
```typescript
// Frontend (SvelteKit)
const { data, error } = await supabase.auth.getClaims()
// → 로컬 검증, ~1-5ms (네트워크 불필요)

// vs getUser()
const { data, error } = await supabase.auth.getUser()
// → 서버 검증, ~100-300ms (네트워크 필요)
```

### Backend JWT 검증
```python
# Supabase Python SDK - get_claims() 사용 (권장)
# JWKS 자동 캐싱 + 로컬 검증으로 get_user()보다 빠름
response = supabase.auth.get_claims()

# 특정 JWT 검증
response = supabase.auth.get_claims(jwt=token)
```

## 3. Key Types 정리

> [!CAUTION]
> **절대 사용 금지: Legacy Key 이름**
> - ❌ `ANON` → ✅ `PUBLIC` (또는 `PUBLISHABLE`)
> - ❌ `SERVICE_ROLE` → ✅ `SECRET`
> - Legacy 이름 사용 시 혼란 + 향후 호환성 문제 발생

> [!WARNING]
> **Legacy JWT 키 지원 종료 (2025.10 이후 점진적 중단)**
> - `eyJ...` 포맷의 Legacy anon/service_role 키는 deprecated
> - 신규 프로젝트는 새 포맷만 사용: `sb_publishable_...`, `sb_secret_...`

| Key Type | 새 이름 | ~~Legacy 이름~~ | Format | 용도 |
|----------|---------|-----------------|--------|------|
| **Publishable Key** | `PUBLIC_KEY` | ~~ANON~~ | `sb_publishable_...` | Frontend 초기화 |
| **Secret Key** | `SECRET_KEY` | ~~SERVICE_ROLE~~ | `sb_secret_...` | Backend RLS 우회 |
| **JWT Signing Key** | - | - | RSA Public Key | JWT 서명 검증 (JWKS) |

### 환경변수 네이밍 규칙
```bash
# ✅ 올바른 네이밍
SUPABASE_PUBLIC_KEY=sb_publishable_...
SUPABASE_SECRET_KEY=sb_secret_...

# ❌ 절대 사용 금지 (Legacy)
SUPABASE_ANON_KEY=eyJ...        # 이거 쓰면 안됨
SUPABASE_SERVICE_ROLE_KEY=eyJ... # 이것도 쓰면 안됨
```

## 4. RBAC 권한 검증

> [!IMPORTANT]
> **Service Client로는 `has_role` RPC 사용 불가**
> - `has_role`은 내부적으로 `auth.uid()` 사용
> - Service Client는 유저 컨텍스트가 없어서 `auth.uid() = NULL`

### Backend (FastAPI) - 올바른 방법
```python
async def require_admin(jwt_token: str):
    supabase = get_supabase_service_client()
    
    # 1. JWT에서 user_id 추출 (JWKS 사용, 로컬 검증)
    claims = supabase.auth.get_claims(jwt=jwt_token)
    user_id = claims.data["sub"]
    
    # 2. Service Client로 직접 user_roles 쿼리
    result = supabase.from_("user_roles") \
        .select("role_id") \
        .eq("user_id", user_id) \
        .eq("role_id", "admin") \
        .execute()
    
    if not result.data:
        raise HTTPException(403, "Admin required")
```

### Frontend (SvelteKit) - `has_role` RPC 사용 가능
```typescript
// User Client는 auth.uid() 컨텍스트가 있음
const isAdmin = await locals.auth.hasRole('admin')
```

## 5. Token Refresh & Error Handling

### Token Refresh (Frontend)
```typescript
// SvelteKit - hooks.server.ts에서 자동 갱신
// @supabase/ssr이 쿠키 기반으로 세션 관리
const { data: { session } } = await supabase.auth.getSession()
// → Access Token 만료 시 Refresh Token으로 자동 갱신
```

### 표준 Error 코드
| HTTP Status | 의미 | 처리 |
|-------------|------|------|
| 401 | JWT 만료/무효 | Frontend: 로그인 페이지 리다이렉트 |
| 403 | 권한 부족 | "접근 권한이 없습니다" 메시지 |
| 500 | 서버 오류 | 로그 확인, 재시도 |

## 6. 보안 체크리스트

- [ ] `SUPABASE_SECRET_KEY`는 서버 환경변수로만 관리
- [ ] Service Client는 Backend에서만 사용
- [ ] Frontend에서 Secret Key 절대 노출 금지
- [ ] 민감한 Admin 작업에 Audit Logging 구현
- [ ] RLS 정책은 Frontend 보호를 위해 유지
