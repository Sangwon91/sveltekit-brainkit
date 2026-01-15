# Cache Optimization

캐시의 효과를 측정하고 최적화하기 위한 가이드입니다.

---

## 캐시 히트율 측정

```typescript
// src/lib/server/cache/metrics.ts
interface CacheMetrics {
  hits: number;
  misses: number;
  errors: number;
}

class CacheMetricsCollector {
  private metrics = new Map<string, CacheMetrics>();
  
  recordHit(key: string): void {
    const m = this.metrics.get(key) || { hits: 0, misses: 0, errors: 0 };
    m.hits++;
    this.metrics.set(key, m);
  }
  
  recordMiss(key: string): void {
    const m = this.metrics.get(key) || { hits: 0, misses: 0, errors: 0 };
    m.misses++;
    this.metrics.set(key, m);
  }
  
  recordError(key: string): void {
    const m = this.metrics.get(key) || { hits: 0, misses: 0, errors: 0 };
    m.errors++;
    this.metrics.set(key, m);
  }
  
  getHitRate(key: string): number {
    const m = this.metrics.get(key);
    if (!m || m.hits + m.misses === 0) return 0;
    return m.hits / (m.hits + m.misses);
  }
  
  getAllMetrics(): Map<string, CacheMetrics> {
    return new Map(this.metrics);
  }
  
  reset(): void {
    this.metrics.clear();
  }
}

export const cacheMetrics = new CacheMetricsCollector();
```

```typescript
// src/lib/server/cache/index.ts (수정)
import { cacheMetrics } from './metrics';

// 캐시 어댑터를 래핑하여 메트릭 수집
export const cacheWithMetrics: CacheAdapter = {
  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await cache.get<T>(key);
      if (value !== null) {
        cacheMetrics.recordHit(key);
      } else {
        cacheMetrics.recordMiss(key);
      }
      return value;
    } catch (error) {
      cacheMetrics.recordError(key);
      throw error;
    }
  },
  
  async set(key: string, value: unknown, ttl?: number): Promise<void> {
    return cache.set(key, value, ttl);
  },
  
  async delete(key: string): Promise<void> {
    return cache.delete(key);
  },
  
  async clear(): Promise<void> {
    return cache.clear();
  }
};
```

---

## TTL 선택 가이드라인

| 데이터 유형 | 권장 TTL | 이유 |
|------------|---------|------|
| 정적 콘텐츠 (블로그 포스트) | 1시간 ~ 1일 | 거의 변경되지 않음 |
| 동적 콘텐츠 (제품 목록) | 5분 ~ 1시간 | 자주 변경되지만 즉시 반영 불필요 |
| 사용자 프로필 | 5분 ~ 15분 | 개인화된 데이터, 보안 고려 |
| 실시간 데이터 (주식 가격) | 10초 ~ 1분 | 매우 자주 변경됨 |
| 집계 데이터 (통계) | 1시간 ~ 1일 | 계산 비용이 높음 |

---

## 캐시 워밍업

애플리케이션 시작 시 자주 사용되는 데이터를 미리 캐시에 로드합니다.

```typescript
// src/lib/server/cache/warmup.ts
import { cache } from './index';
import { cacheKeys } from './keys';
import { db } from './db';

/**
 * 애플리케이션 시작 시 캐시를 워밍업합니다.
 */
export async function warmupCache(): Promise<void> {
  try {
    // 자주 사용되는 데이터를 미리 로드
    const [products, categories, featuredPosts] = await Promise.all([
      db.getProducts(),
      db.getCategories(),
      db.getFeaturedPosts()
    ]);
    
    await Promise.all([
      cache.set(cacheKeys.products.list(), products, 300),
      cache.set('app:categories', categories, 600),
      cache.set('app:featured-posts', featuredPosts, 1800)
    ]);
    
    console.log('Cache warmed up successfully');
  } catch (error) {
    console.error('Failed to warm up cache:', error);
    // 워밍업 실패는 애플리케이션 시작을 막지 않음
  }
}
```

```typescript
// src/hooks.server.ts
import { warmupCache } from '$lib/server/cache/warmup';

export async function handle({ event, resolve }) {
  // 프로덕션 환경에서만 워밍업
  if (process.env.NODE_ENV === 'production') {
    warmupCache().catch(console.error);
  }
  
  return resolve(event);
}
```

---

## 개발 환경 vs 프로덕션 환경

```typescript
// src/lib/server/cache/index.ts (개선)
import { env } from '$env/dynamic/private';
import { dev } from '$app/environment';
import { MemoryCache } from './memory';
import { RedisCache } from './redis';
import type { CacheAdapter } from './types';

let cacheAdapter: CacheAdapter;

if (dev) {
  // 개발 환경: 항상 Memory 캐시 사용 (Redis 없이도 개발 가능)
  console.log('Using MemoryCache for development');
  cacheAdapter = new MemoryCache();
} else {
  // 프로덕션: Redis 사용 (환경 변수로 설정)
  if (env.REDIS_URL) {
    const Redis = (await import('ioredis')).default;
    const redis = new Redis(env.REDIS_URL);
    cacheAdapter = new RedisCache(redis);
    console.log('Using RedisCache for production');
  } else {
    console.warn('REDIS_URL not set, falling back to MemoryCache');
    cacheAdapter = new MemoryCache();
  }
}

export const cache = cacheAdapter;
```

