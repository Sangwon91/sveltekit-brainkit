---
trigger: always_on
glob: "*"
description: "SvelteKit v5 Caching Guidelines"
---

# Caching Guidelines

## Philosophy

**Cache Strategically, Invalidate Explicitly.**

Caching is a performance optimization that must be balanced with data freshness and security. Use HTTP cache headers for CDN/browser caching, SvelteKit's `depends`/`invalidate` for application-level cache control, and key-value stores for server-side caching. Always consider user-specific data security when caching.

---

## Quick Reference

### HTTP Cache Headers

```typescript
// 정적 콘텐츠: CDN 1시간, 브라우저 5분
setHeaders({ 'cache-control': 'public, max-age=300, s-maxage=3600' });

// 사용자별 데이터: 브라우저만 5분
setHeaders({ 'cache-control': 'private, max-age=300' });

// 실시간 데이터: 캐시 비활성화
setHeaders({ 'cache-control': 'no-store' });

// 재검증 허용 (ETag와 함께)
setHeaders({ 'cache-control': 'no-cache' });
```

### SvelteKit Cache Invalidation

```typescript
// Load 함수에서 태그 설정
export const load = async ({ depends }) => {
  depends('app:products');
  return { products: await db.getProducts() };
};

// Form Action에서 무효화
export const actions = {
  create: async ({ request }) => {
    await db.createProduct(/* ... */);
    await invalidate('app:products'); // 태그 기반
    await invalidate('/products');    // URL 기반
  }
};
```

### Server-Side Cache Pattern

```typescript
// 캐시 조회 (폴백 포함)
let products = await cache.get<Product[]>(cacheKey);
if (!products) {
  products = await db.getProducts();
  await cache.set(cacheKey, products, 300); // TTL: 5분
}
```

---

## Core Knowledge (Cheat Sheet)

### ✅ Do's
- **Use `setHeaders` in load functions**: Control HTTP cache headers for HTML and data responses.
- **Use `depends()` for cache tagging**: Tag your data dependencies for granular invalidation.
- **Use `invalidate()` for explicit cache busting**: Invalidate by tag or URL when data changes.
- **Use `private` cache for user-specific data**: Prevent CDN caching of personalized content.
- **Use `stale-while-revalidate` for UX**: Serve stale content while fetching fresh data.
- **Abstract cache implementation**: Use interfaces/protocols to decouple from specific cache backends.
- **Handle cache failures gracefully**: Cache is an optimization, not a requirement. Always have a fallback.
- **Invalidate both SvelteKit and server-side cache**: When data changes, invalidate all cache layers.
- **Use consistent cache key strategies**: Use `cacheKeys` utility for all cache key generation.
- **Monitor cache performance**: Track hit rates and optimize TTL values.

### ❌ Don'ts
- **Don't cache sensitive user data publicly**: Always use `private` for personalized responses.
- **Don't ignore browser cache conflicts**: Test cache behavior in both dev and production.
- **Don't hardcode cache keys**: Use consistent key generation strategies via `cacheKeys` utility.
- **Don't cache without invalidation strategy**: Plan how to invalidate when data changes (both SvelteKit and server-side).
- **Don't include sensitive data in cache keys**: Never put passwords, tokens, or PII in cache keys.
- **Don't ignore cache errors**: Always implement error handling and fallback strategies.
- **Don't use write-behind for critical data**: Use write-through for financial or transactional data.
- **Don't skip cache key normalization**: Always validate and normalize user input before using in cache keys.

---

## Tech Stack

- **SvelteKit v5** (load functions, `depends`, `invalidate`, `setHeaders`)
- **FastAPI** (Response headers, cache middleware)
- **Cache Backends**: Redis, Memory (abstracted)

> [!NOTE]
> For basic HTTP cache header patterns, see [Cache Patterns Reference](frontend-architecture/references/cache_patterns.md).

---

## 1. HTML 캐시

SvelteKit의 HTML 응답은 기본적으로 캐시되지 않습니다. `setHeaders`를 사용하여 HTML 캐시를 명시적으로 제어할 수 있습니다.

### 정적 페이지 캐싱

```typescript
// src/routes/blog/[slug]/+page.server.ts
export const load: PageServerLoad = async ({ setHeaders, params }) => {
  const post = await db.getPost(params.slug);
  
  // HTML을 CDN에서 1시간, 브라우저에서 5분간 캐시
  setHeaders({
    'cache-control': 'public, max-age=300, s-maxage=3600'
  });
  
  return { post };
};
```

### 동적 페이지 (캐시 비활성화)

```typescript
// src/routes/dashboard/+page.server.ts
export const load: PageServerLoad = async ({ setHeaders, locals }) => {
  const userData = await db.getUserData(locals.user.id);
  
  // HTML 캐시 비활성화 (개인화된 콘텐츠)
  setHeaders({
    'cache-control': 'private, no-cache, no-store, must-revalidate'
  });
  
  return { userData };
};
```

### CDN vs 브라우저 캐시

| 헤더 값 | CDN 캐시 | 브라우저 캐시 | 사용 사례 |
|---------|----------|---------------|-----------|
| `public, max-age=60, s-maxage=3600` | 1시간 | 1분 | 블로그 포스트 |
| `private, max-age=300` | ❌ | 5분 | 사용자 대시보드 |
| `no-store` | ❌ | ❌ | 실시간 데이터 |

> [!TIP]
> `s-maxage`는 CDN/프록시 캐시를 위한 값이며, `max-age`는 브라우저 캐시를 위한 값입니다.

---

## 2. SvelteKit depends & invalidate

SvelteKit의 `depends`와 `invalidate`는 애플리케이션 레벨 캐시 무효화를 위한 핵심 메커니즘입니다.

### depends() - 캐시 태깅

```typescript
// src/routes/products/+page.server.ts
export const load: PageServerLoad = async ({ depends }) => {
  depends('app:products');
  const products = await db.getProducts();
  return { products };
};
```

### invalidate() - 태그 기반 무효화

```typescript
// src/routes/admin/products/+page.server.ts
import { invalidate } from '$app/navigation';

export const actions = {
  create: async ({ request }) => {
    await db.createProduct(/* ... */);
    await invalidate('app:products'); // 태그 기반
    await invalidate('/products');    // URL 기반
    return { success: true };
  }
};
```

### 계층적 태그 전략

```typescript
// 목록 페이지
depends('app:products');
depends('app:products:list');

// 상세 페이지
depends('app:products');
depends(`app:products:${params.id}`);

// 무효화 시
await invalidate('app:products');        // 모든 products 관련
await invalidate('app:products:list');   // 목록만
await invalidate(`app:products:${id}`);  // 특정 상품만
```

---

## 3. 브라우저 캐시 간섭 문제

HTTP 캐시 헤더와 SvelteKit의 내부 캐시 메커니즘이 충돌할 수 있습니다.

### 해결 방법

1. **HTML은 no-cache, 데이터만 캐시**: HTML 자체는 항상 최신 버전을 가져오고, 데이터만 캐시
2. **개발 환경에서 캐시 비활성화**: 개발 중에는 캐시를 완전히 비활성화
3. **ETag 활용**: 콘텐츠가 변경되었을 때만 새 버전을 가져옴

> [!NOTE]
> 상세 내용은 [Browser Cache Interference](svelte-cache/references/browser_cache_interference.md) 참조

---

## 4. Server-Side Cache (Key-Value)

서버 측에서 key-value 캐시를 사용할 때는 구현을 추상화하여 Redis, Memory, 또는 다른 백엔드로 쉽게 전환할 수 있도록 합니다.

### 기본 패턴

```typescript
// 캐시 조회 (폴백 포함)
const cacheKey = cacheKeys.products.list();
let products = await cache.get<Product[]>(cacheKey);

if (!products) {
  products = await db.getProducts();
  await cache.set(cacheKey, products, 300); // TTL: 5분
}
```

### 캐시 키 전략

```typescript
// src/lib/server/cache/keys.ts
export const cacheKeys = {
  products: {
    list: (filters?: string) => `app:products:list:${filters || 'all'}`,
    byId: (id: string) => `app:products:${id}`,
  },
  user: {
    dashboard: (userId: string) => `app:user:${userId}:dashboard`,
  }
};
```

> [!WARNING]
> 캐시 키에 사용자 입력을 직접 사용하지 마세요. 항상 `normalizeCacheKey`를 통해 정규화하거나, 검증된 값만 사용하세요.

> [!NOTE]
> 상세 구현은 [Server-Side Caching](svelte-cache/references/server_side_caching.md) 참조

---

## 5. Cache Invalidation 연동

SvelteKit의 `invalidate()`는 애플리케이션 레벨 캐시만 무효화합니다. 서버 사이드 key-value 캐시(Redis 등)도 함께 무효화해야 합니다.

### 기본 패턴

```typescript
export const actions = {
  create: async ({ request }) => {
    await db.createProduct(/* ... */);
    
    // 1. SvelteKit 캐시 무효화
    await invalidate('app:products');
    
    // 2. 서버 사이드 캐시 무효화
    await cache.delete(cacheKeys.products.list());
    
    return { success: true };
  }
};
```

> [!NOTE]
> 상세 내용은 [Cache Invalidation](svelte-cache/references/cache_invalidation.md) 참조

---

## 6. Cache Headers: no-cache vs no-store

| 헤더 | 의미 | 사용 사례 |
|------|------|-----------|
| `no-cache` | 캐시 가능하지만 항상 서버에 재검증 | 사용자별 데이터 (ETag와 함께) |
| `no-store` | 캐시 금지 (완전 비활성화) | 민감한 정보, 실시간 데이터 |

```typescript
// no-cache: 재검증 허용
setHeaders({ 'cache-control': 'no-cache' });

// no-store: 완전 비활성화
setHeaders({ 'cache-control': 'no-store' });
```

---

## 7. stale-while-revalidate (SWR)

캐시된 데이터를 즉시 반환하면서 백그라운드에서 새 데이터를 가져오는 패턴입니다.

```typescript
// max-age=60: 60초간 fresh
// stale-while-revalidate=300: fresh 기간 이후 300초간 stale 허용
setHeaders({
  'cache-control': 'public, max-age=60, stale-while-revalidate=300'
});
```

---

## 8. 사용자별 캐시 보안

사용자별 데이터를 캐싱할 때는 보안을 최우선으로 고려해야 합니다.

### 기본 원칙

- **private 캐시 사용**: 사용자별 데이터는 `private`로 설정하여 CDN 캐시 방지
- **사용자 ID 기반 키**: 캐시 키에 사용자 ID를 포함하여 사용자별로 분리
- **캐시 키 검증**: 사용자 입력을 직접 사용하지 않고 정규화

```typescript
// ✅ GOOD
setHeaders({ 'cache-control': 'private, max-age=300' });
const cacheKey = cacheKeys.user.dashboard(userId);

// ❌ BAD
setHeaders({ 'cache-control': 'public, max-age=3600' }); // CDN 노출 위험
const cacheKey = `user:${userInput}`; // 입력 검증 없음
```

> [!NOTE]
> 상세 내용은 [Cache Security](svelte-cache/references/cache_security.md) 참조

---

## 9. Write Patterns

캐시와 데이터베이스 간의 쓰기 전략에는 두 가지 주요 패턴이 있습니다.

| 패턴 | 장점 | 단점 | 사용 사례 |
|------|------|------|-----------|
| **Write-Through** | 데이터 일관성 보장, 안전함 | 쓰기 성능 느림 | 금융 데이터, 중요한 트랜잭션 |
| **Write-Behind** | 쓰기 성능 빠름, 사용자 경험 좋음 | 데이터 손실 위험, 복잡함 | 조회수, 좋아요, 로그 데이터 |

> [!WARNING]
> Write-behind 패턴은 서버 재시작 시 큐에 남은 데이터가 손실될 수 있습니다. 중요한 데이터에는 사용하지 마세요. 프로덕션 환경에서는 영속적 큐(Redis Streams, RabbitMQ 등)를 사용하세요.

> [!NOTE]
> 상세 내용은 [Write Patterns](svelte-cache/references/write_patterns.md) 참조

---

## Decision Tree

```
데이터를 캐시해야 하나?
├── Yes
│   ├── 사용자별 데이터?
│   │   ├── Yes → private 캐시 + 사용자 ID 기반 키 + Vary 헤더
│   │   │         + 짧은 TTL (5-15분) + 로그아웃 시 무효화
│   │   └── No → public 캐시 가능
│   │
│   ├── 실시간 데이터?
│   │   ├── Yes → no-store 또는 no-cache + 매우 짧은 TTL (10초-1분)
│   │   └── No → 적절한 max-age 설정 (5분-1시간)
│   │
│   ├── 변경 빈도는?
│   │   ├── 거의 변경 안됨 → 긴 TTL (1시간-1일) + stale-while-revalidate
│   │   ├── 가끔 변경 → 중간 TTL (5-30분) + invalidate() 사용
│   │   └── 자주 변경 → 짧은 TTL (1-5분) 또는 no-cache
│   │
│   ├── 트래픽이 많은가?
│   │   ├── Yes → stale-while-revalidate + CDN 캐시 활용
│   │   └── No → 기본 캐시 전략
│   │
│   └── 쓰기 작업?
│       ├── 중요 데이터 (금융, 트랜잭션) → Write-Through
│       ├── 비중요 데이터 (조회수, 로그) → Write-Behind (Redis Queue 사용)
│       └── 일반 데이터 → Write-Through 또는 캐시 무효화
│
└── No → 캐시하지 않음
    └── 민감한 정보, 일회성 데이터, 매우 큰 데이터
```

### 빠른 참조: TTL 선택 가이드

| 데이터 유형 | TTL | HTTP 헤더 | 캐시 타입 |
|------------|-----|-----------|----------|
| 정적 콘텐츠 | 1시간-1일 | `public, max-age=3600` | public |
| 사용자 프로필 | 5-15분 | `private, max-age=300` | private |
| 상품 목록 | 5-30분 | `public, max-age=300, stale-while-revalidate=600` | public |
| 실시간 데이터 | 10초-1분 | `public, max-age=10, must-revalidate` | public |
| 민감한 정보 | 캐시 안함 | `no-store` | - |

---

## References

- [Server-Side Caching](svelte-cache/references/server_side_caching.md) - Key-value 캐시 구현 및 추상화
- [Cache Invalidation](svelte-cache/references/cache_invalidation.md) - SvelteKit과 서버 캐시 무효화 연동
- [Write Patterns](svelte-cache/references/write_patterns.md) - Write-through, Write-behind 패턴
- [Cache Security](svelte-cache/references/cache_security.md) - 사용자별 캐시 보안 및 캐시 오염 방지
- [Cache Optimization](svelte-cache/references/cache_optimization.md) - 성능 최적화 및 모니터링
- [Browser Cache Interference](svelte-cache/references/browser_cache_interference.md) - 브라우저 캐시 간섭 문제 해결
- [Cache Patterns Reference](frontend-architecture/references/cache_patterns.md) - Basic HTTP cache headers
- [Data Fetching Guidelines](data-fetching.md) - SvelteKit load functions
- [Backend Architecture](backend-architecture.md) - FastAPI patterns
