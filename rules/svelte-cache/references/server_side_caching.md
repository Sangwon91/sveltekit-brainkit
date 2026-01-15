# Server-Side Caching

서버 측에서 key-value 캐시를 사용할 때는 구현을 추상화하여 Redis, Memory, 또는 다른 백엔드로 쉽게 전환할 수 있도록 합니다.

---

## 캐시 인터페이스 정의

```typescript
// src/lib/server/cache/types.ts

export class CacheError extends Error {
  constructor(message: string, public readonly cause?: unknown) {
    super(message);
    this.name = 'CacheError';
  }
}

export interface CacheAdapter {
  get<T>(key: string): Promise<T | null>;
  set(key: string, value: unknown, ttl?: number): Promise<void>;
  delete(key: string): Promise<void>;
  clear(): Promise<void>;
}
```

---

## Memory 캐시 구현

```typescript
// src/lib/server/cache/memory.ts
import type { CacheAdapter } from './types';
import { CacheError } from './types';

export class MemoryCache implements CacheAdapter {
  private cache = new Map<string, { value: unknown; expires?: number }>();
  
  async get<T>(key: string): Promise<T | null> {
    try {
      const item = this.cache.get(key);
      if (!item) return null;
      
      if (item.expires && item.expires < Date.now()) {
        this.cache.delete(key);
        return null;
      }
      
      return item.value as T;
    } catch (error) {
      throw new CacheError(`Failed to get cache key: ${key}`, error);
    }
  }
  
  async set(key: string, value: unknown, ttl?: number): Promise<void> {
    try {
      const expires = ttl ? Date.now() + ttl * 1000 : undefined;
      this.cache.set(key, { value, expires });
    } catch (error) {
      throw new CacheError(`Failed to set cache key: ${key}`, error);
    }
  }
  
  async delete(key: string): Promise<void> {
    try {
      this.cache.delete(key);
    } catch (error) {
      throw new CacheError(`Failed to delete cache key: ${key}`, error);
    }
  }
  
  async clear(): Promise<void> {
    try {
      this.cache.clear();
    } catch (error) {
      throw new CacheError('Failed to clear cache', error);
    }
  }
}
```

---

## Redis 캐시 구현

```typescript
// src/lib/server/cache/redis.ts
import type { CacheAdapter } from './types';
import { CacheError } from './types';
import { Redis } from 'ioredis';

export class RedisCache implements CacheAdapter {
  constructor(private redis: Redis) {}
  
  async get<T>(key: string): Promise<T | null> {
    try {
      const value = await this.redis.get(key);
      if (!value) return null;
      
      try {
        return JSON.parse(value) as T;
      } catch (parseError) {
        // JSON 파싱 실패 시 캐시 키 삭제 (손상된 데이터)
        await this.redis.del(key).catch(() => {});
        throw new CacheError(`Failed to parse cached value for key: ${key}`, parseError);
      }
    } catch (error) {
      if (error instanceof CacheError) throw error;
      throw new CacheError(`Failed to get cache key: ${key}`, error);
    }
  }
  
  async set(key: string, value: unknown, ttl?: number): Promise<void> {
    try {
      const serialized = JSON.stringify(value);
      if (ttl) {
        await this.redis.setex(key, ttl, serialized);
      } else {
        await this.redis.set(key, serialized);
      }
    } catch (error) {
      throw new CacheError(`Failed to set cache key: ${key}`, error);
    }
  }
  
  async delete(key: string): Promise<void> {
    try {
      await this.redis.del(key);
    } catch (error) {
      throw new CacheError(`Failed to delete cache key: ${key}`, error);
    }
  }
  
  async clear(): Promise<void> {
    try {
      await this.redis.flushdb();
    } catch (error) {
      throw new CacheError('Failed to clear cache', error);
    }
  }
}
```

---

## SvelteKit에서의 사용

```typescript
// src/lib/server/cache/index.ts
import { env } from '$env/dynamic/private';
import { dev } from '$app/environment';
import { MemoryCache } from './memory';
import { RedisCache } from './redis';
import type { CacheAdapter } from './types';

let cacheAdapter: CacheAdapter;

if (dev) {
  // 개발 환경: 항상 Memory 캐시 사용
  cacheAdapter = new MemoryCache();
} else {
  // 프로덕션: Redis 사용 (환경 변수로 설정)
  if (env.REDIS_URL) {
    const Redis = (await import('ioredis')).default;
    const redis = new Redis(env.REDIS_URL);
    cacheAdapter = new RedisCache(redis);
  } else {
    cacheAdapter = new MemoryCache();
  }
}

export const cache = cacheAdapter;
```

```typescript
// src/routes/products/+page.server.ts
import type { PageServerLoad } from './$types';
import { cache } from '$lib/server/cache';
import { cacheKeys } from '$lib/server/cache/keys';
import { CacheError } from '$lib/server/cache/types';
import type { Product } from '$lib/types';

export const load: PageServerLoad = async () => {
  const cacheKey = cacheKeys.products.list();
  
  // 캐시에서 조회 (에러 발생 시 DB로 폴백)
  let products: Product[] | null = null;
  
  try {
    products = await cache.get<Product[]>(cacheKey);
  } catch (error) {
    // 캐시 오류는 로깅하되, 애플리케이션은 계속 동작
    console.error('Cache get error:', error);
  }
  
  if (!products) {
    // 캐시 미스 또는 캐시 오류: DB에서 조회
    products = await db.getProducts();
    
    // 캐시에 저장 시도 (실패해도 애플리케이션은 계속 동작)
    try {
      await cache.set(cacheKey, products, 300); // TTL: 5분
    } catch (error) {
      console.error('Cache set error:', error);
    }
  }
  
  return { products };
};
```

---

## 캐시 키 전략

```typescript
// src/lib/server/cache/keys.ts

/**
 * 캐시 키에 사용할 수 없는 문자를 정규화합니다.
 * 보안을 위해 사용자 입력을 캐시 키에 직접 사용하지 마세요.
 */
export function normalizeCacheKey(input: string): string {
  // 허용된 문자만 사용, 특수문자 제거
  return input.replace(/[^a-zA-Z0-9-]/g, '-').toLowerCase();
}

/**
 * 일관된 캐시 키 생성 함수들
 * 모든 캐시 키는 이 함수들을 통해 생성해야 합니다.
 */
export const cacheKeys = {
  // 네임스페이스 접두사로 충돌 방지
  prefix: 'app',
  
  products: {
    list: (filters?: string) => {
      const filterKey = filters ? normalizeCacheKey(filters) : 'all';
      return `${cacheKeys.prefix}:products:list:${filterKey}`;
    },
    byId: (id: string) => `${cacheKeys.prefix}:products:${normalizeCacheKey(id)}`,
    byCategory: (category: string) => 
      `${cacheKeys.prefix}:products:category:${normalizeCacheKey(category)}`
  },
  user: {
    byId: (id: string) => `${cacheKeys.prefix}:user:${normalizeCacheKey(id)}`,
    profile: (id: string) => `${cacheKeys.prefix}:user:${normalizeCacheKey(id)}:profile`,
    dashboard: (id: string) => `${cacheKeys.prefix}:user:${normalizeCacheKey(id)}:dashboard`,
    settings: (id: string) => `${cacheKeys.prefix}:user:${normalizeCacheKey(id)}:settings`
  }
};
```

> [!WARNING]
> 캐시 키에 사용자 입력을 직접 사용하지 마세요. 항상 `normalizeCacheKey`를 통해 정규화하거나, 검증된 값만 사용하세요. 민감한 정보(비밀번호, 토큰 등)는 절대 캐시 키에 포함하지 마세요.

---

## FastAPI에서의 사용 (Python)

```python
# backend/apps/api/src/app/cache/protocol.py
from typing import Protocol, Optional, TypeVar

T = TypeVar('T')

class CacheAdapter(Protocol):
    async def get(self, key: str) -> Optional[T]: ...
    async def set(self, key: str, value: T, ttl: Optional[int] = None) -> None: ...
    async def delete(self, key: str) -> None: ...
    async def clear(self) -> None: ...
```

```python
# backend/apps/api/src/app/cache/redis_cache.py
from typing import Optional, TypeVar
import json
from redis import Redis

T = TypeVar('T')

class RedisCache:
    def __init__(self, redis: Redis):
        self.redis = redis
    
    async def get(self, key: str) -> Optional[T]:
        value = await self.redis.get(key)
        return json.loads(value) if value else None
    
    async def set(self, key: str, value: T, ttl: Optional[int] = None) -> None:
        serialized = json.dumps(value)
        if ttl:
            await self.redis.setex(key, ttl, serialized)
        else:
            await self.redis.set(key, serialized)
    
    async def delete(self, key: str) -> None:
        await self.redis.delete(key)
    
    async def clear(self) -> None:
        await self.redis.flushdb()
```

