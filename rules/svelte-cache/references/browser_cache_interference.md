# Browser Cache Interference

HTTP 캐시 헤더와 SvelteKit의 내부 캐시 메커니즘이 충돌할 수 있습니다. 특히 개발 환경과 프로덕션 환경에서 다르게 동작할 수 있습니다.

---

## 문제 상황

브라우저가 HTML을 캐시하면, SvelteKit의 `invalidate()`가 실행되어도 브라우저는 캐시된 HTML을 사용하여 새로운 데이터를 로드하지 않을 수 있습니다.

```typescript
// 문제가 될 수 있는 설정
setHeaders({
  'cache-control': 'public, max-age=3600' // HTML을 1시간 캐시
});
```

---

## 해결 방법 1: HTML은 no-cache, 데이터만 캐시

HTML 자체는 항상 최신 버전을 가져오고, 데이터만 캐시합니다.

```typescript
// src/routes/products/+page.server.ts
export const load: PageServerLoad = async ({ setHeaders }) => {
  const products = await db.getProducts();
  
  // HTML은 캐시하지 않음 (항상 최신)
  setHeaders({
    'cache-control': 'no-cache'
  });
  
  // 데이터는 JSON으로 캐시 가능 (별도 엔드포인트)
  return { products };
};
```

---

## 해결 방법 2: 개발 환경에서 캐시 비활성화

개발 중에는 캐시를 완전히 비활성화하여 예측 가능한 동작을 보장합니다.

```typescript
// src/routes/products/+page.server.ts
import { dev } from '$app/environment';

export const load: PageServerLoad = async ({ setHeaders }) => {
  const products = await db.getProducts();
  
  if (dev) {
    // 개발 환경: 캐시 완전 비활성화
    setHeaders({
      'cache-control': 'no-store, no-cache, must-revalidate'
    });
  } else {
    // 프로덕션: 적절한 캐시 설정
    setHeaders({
      'cache-control': 'public, max-age=300, s-maxage=3600'
    });
  }
  
  return { products };
};
```

---

## 해결 방법 3: ETag 활용

ETag를 사용하여 콘텐츠가 변경되었을 때만 새 버전을 가져옵니다.

> [!NOTE]
> SvelteKit의 `load` 함수에서는 `Response`를 직접 반환할 수 없습니다. ETag 검증은 서버 측에서 처리하거나, 별도의 API 엔드포인트를 사용해야 합니다.

```typescript
// src/lib/server/utils/etag.ts
import { createHash } from 'crypto';

/**
 * 데이터의 ETag를 생성합니다.
 * SHA-256 해시를 사용하여 충돌 가능성을 최소화합니다.
 */
export function generateETag(data: unknown): string {
  const serialized = JSON.stringify(data);
  return `"${createHash('sha256').update(serialized).digest('hex').substring(0, 16)}"`;
}

/**
 * ETag 비교 (약한 검증)
 */
export function compareETags(etag1: string, etag2: string): boolean {
  // ETag는 따옴표로 감싸져 있을 수 있음
  const normalize = (etag: string) => etag.replace(/^"|"$/g, '').toLowerCase();
  return normalize(etag1) === normalize(etag2);
}
```

```typescript
// src/routes/products/+page.server.ts
import type { PageServerLoad } from './$types';
import { generateETag, compareETags } from '$lib/server/utils/etag';

export const load: PageServerLoad = async ({ setHeaders, request }) => {
  const products = await db.getProducts();
  const etag = generateETag(products);
  
  // 클라이언트가 보낸 If-None-Match 헤더 확인
  const ifNoneMatch = request.headers.get('if-none-match');
  
  if (ifNoneMatch && compareETags(ifNoneMatch, etag)) {
    // 데이터가 변경되지 않았음 - 304 응답
    // SvelteKit에서는 setHeaders로 상태 코드를 설정할 수 없으므로,
    // 별도의 API 엔드포인트를 사용하거나 서버 미들웨어에서 처리해야 합니다.
    setHeaders({
      'etag': etag,
      'cache-control': 'public, max-age=300'
    });
    
    // 데이터는 반환하되, 클라이언트가 캐시를 사용하도록 유도
    return { products, _cached: true };
  }
  
  setHeaders({
    'etag': etag,
    'cache-control': 'public, max-age=300'
  });
  
  return { products };
};
```

> [!TIP]
> 완전한 ETag 검증을 위해서는 FastAPI 백엔드에서 처리하거나, SvelteKit의 서버 미들웨어를 사용하는 것이 좋습니다.

---

## 개발 환경 디버깅

개발 중 브라우저 캐시 문제를 디버깅하려면:

1. **브라우저 DevTools**: Network 탭에서 "Disable cache" 체크
2. **Hard Refresh**: `Ctrl+Shift+R` (Windows/Linux) 또는 `Cmd+Shift+R` (Mac)
3. **시크릿 모드**: 캐시 없이 테스트

