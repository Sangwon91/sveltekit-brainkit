# Cache Invalidation

SvelteKit의 `invalidate()`는 애플리케이션 레벨 캐시만 무효화합니다. 서버 사이드 key-value 캐시(Redis 등)도 함께 무효화해야 합니다.

---

## Form Action에서 서버 캐시 무효화

```typescript
// src/routes/admin/products/+page.server.ts
import { invalidate } from '$app/navigation';
import { cache } from '$lib/server/cache';
import { cacheKeys } from '$lib/server/cache/keys';
import type { Actions } from './$types';

export const actions: Actions = {
  create: async ({ request }) => {
    const formData = await request.formData();
    const productData = Object.fromEntries(formData);
    
    // 1. 데이터베이스에 생성
    const newProduct = await db.createProduct(productData);
    
    // 2. SvelteKit 애플리케이션 캐시 무효화
    await invalidate('app:products');
    await invalidate('app:products:list');
    
    // 3. 서버 사이드 캐시 무효화
    try {
      // 관련된 모든 캐시 키 삭제
      await cache.delete(cacheKeys.products.list());
      await cache.delete(cacheKeys.products.list('all'));
    } catch (error) {
      console.error('Failed to invalidate server cache:', error);
      // 캐시 무효화 실패는 로깅하되, 요청은 성공으로 처리
    }
    
    return { success: true, product: newProduct };
  },
  
  update: async ({ request, params }) => {
    const formData = await request.formData();
    const updates = Object.fromEntries(formData);
    
    // 1. 데이터베이스 업데이트
    const updatedProduct = await db.updateProduct(params.id, updates);
    
    // 2. SvelteKit 캐시 무효화
    await invalidate('app:products');
    await invalidate(`app:products:${params.id}`);
    await invalidate(`/products/${params.id}`);
    
    // 3. 서버 사이드 캐시 무효화
    try {
      await cache.delete(cacheKeys.products.byId(params.id));
      await cache.delete(cacheKeys.products.list());
    } catch (error) {
      console.error('Failed to invalidate server cache:', error);
    }
    
    return { success: true, product: updatedProduct };
  },
  
  delete: async ({ params }) => {
    // 1. 데이터베이스 삭제
    await db.deleteProduct(params.id);
    
    // 2. SvelteKit 캐시 무효화
    await invalidate('app:products');
    await invalidate('app:products:list');
    
    // 3. 서버 사이드 캐시 무효화
    try {
      await cache.delete(cacheKeys.products.byId(params.id));
      await cache.delete(cacheKeys.products.list());
    } catch (error) {
      console.error('Failed to invalidate server cache:', error);
    }
    
    return { success: true };
  }
};
```

---

## 캐시 무효화 헬퍼 함수

반복되는 패턴을 헬퍼 함수로 추상화할 수 있습니다.

```typescript
// src/lib/server/cache/invalidation.ts
import { invalidate } from '$app/navigation';
import { cache } from './index';
import { cacheKeys } from './keys';

/**
 * 제품 관련 캐시를 모두 무효화합니다.
 */
export async function invalidateProductCache(productId?: string) {
  // SvelteKit 캐시 무효화
  await invalidate('app:products');
  await invalidate('app:products:list');
  
  if (productId) {
    await invalidate(`app:products:${productId}`);
    await invalidate(`/products/${productId}`);
  }
  
  // 서버 사이드 캐시 무효화
  try {
    await cache.delete(cacheKeys.products.list());
    
    if (productId) {
      await cache.delete(cacheKeys.products.byId(productId));
    }
  } catch (error) {
    console.error('Failed to invalidate server cache:', error);
  }
}

/**
 * 사용자 관련 캐시를 모두 무효화합니다.
 */
export async function invalidateUserCache(userId: string) {
  // SvelteKit 캐시 무효화
  await invalidate(`app:user:${userId}`);
  
  // 서버 사이드 캐시 무효화
  try {
    await cache.delete(cacheKeys.user.byId(userId));
    await cache.delete(cacheKeys.user.profile(userId));
    await cache.delete(cacheKeys.user.dashboard(userId));
    await cache.delete(cacheKeys.user.settings(userId));
  } catch (error) {
    console.error('Failed to invalidate user cache:', error);
  }
}
```

### 사용 예제

```typescript
// src/routes/products/+page.server.ts
import { invalidateProductCache } from '$lib/server/cache/invalidation';

export const actions: Actions = {
  update: async ({ request, params }) => {
    const updates = Object.fromEntries(await request.formData());
    await db.updateProduct(params.id, updates);
    
    // 한 번의 호출로 모든 캐시 무효화
    await invalidateProductCache(params.id);
    
    return { success: true };
  }
};
```

---

## 세션 기반 캐시 무효화

사용자가 로그아웃하거나 권한이 변경되면 관련 캐시를 무효화해야 합니다.

```typescript
// src/routes/auth/logout/+page.server.ts
import { invalidateUserCache } from '$lib/server/cache/invalidation';

export const actions = {
  default: async ({ locals, cookies }) => {
    const userId = locals.user.id;
    
    // 사용자 관련 모든 캐시 무효화
    await invalidateUserCache(userId);
    
    // 세션 쿠키 삭제
    cookies.delete('session');
    
    return redirect('/login');
  }
};
```

