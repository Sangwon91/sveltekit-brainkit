# Write Patterns

캐시와 데이터베이스 간의 쓰기 전략에는 두 가지 주요 패턴이 있습니다: **write-through**와 **write-behind**.

---

## Write-Through 패턴

쓰기 작업을 캐시와 데이터베이스에 **동시에** 수행합니다. 데이터 일관성이 보장되지만 쓰기 성능이 느릴 수 있습니다.

```typescript
// src/lib/server/cache/write-through.ts
import { cache } from './index';
import { db } from './db';

export async function writeThrough<T>(
  cacheKey: string,
  writeFn: () => Promise<T>,
  ttl?: number
): Promise<T> {
  // 1. 데이터베이스에 쓰기
  const result = await writeFn();
  
  // 2. 캐시에 즉시 업데이트
  await cache.set(cacheKey, result, ttl);
  
  return result;
}
```

```typescript
// src/routes/products/+page.server.ts
import { writeThrough } from '$lib/server/cache/write-through';
import { cacheKeys } from '$lib/server/cache/keys';
import { invalidate } from '$app/navigation';

export const actions = {
  update: async ({ request, params }) => {
    const formData = await request.formData();
    const updates = Object.fromEntries(formData);
    
    // Write-through: DB 업데이트와 캐시 업데이트를 동시에
    const updatedProduct = await writeThrough(
      cacheKeys.products.byId(params.id),
      async () => {
        return await db.updateProduct(params.id, updates);
      },
      300 // TTL
    );
    
    // 관련 캐시도 무효화
    await invalidate('app:products');
    
    return { success: true, product: updatedProduct };
  }
};
```

---

## Write-Behind 패턴 (Write-Back)

쓰기 작업을 **캐시에만 먼저** 수행하고, 나중에 비동기로 데이터베이스에 쓰기합니다. 쓰기 성능이 빠르지만 데이터 손실 위험이 있습니다.

> [!WARNING]
> Write-Behind 패턴은 프로덕션 환경에서 신중하게 사용해야 합니다. 메모리 기반 큐는 서버 재시작 시 데이터를 손실할 수 있습니다. 중요한 데이터에는 사용하지 마세요.

```typescript
// src/lib/server/cache/write-behind.ts
import { cache } from './index';
import { CacheError } from './types';

interface PendingWrite {
  key: string;
  value: unknown;
  writeFn: () => Promise<void>;
  retries: number;
  timestamp: number;
}

const MAX_RETRIES = 3;
const FLUSH_INTERVAL_MS = 5000; // 5초
const MAX_QUEUE_SIZE = 1000;

// 영속적 큐를 사용하는 것이 좋습니다 (Redis Streams, RabbitMQ 등)
const writeQueue: PendingWrite[] = [];
let flushTimer: ReturnType<typeof setInterval> | null = null;
let isFlushing = false;

/**
 * 큐를 플러시합니다 (배치 처리)
 */
async function flushQueue(): Promise<void> {
  if (isFlushing || writeQueue.length === 0) return;
  
  isFlushing = true;
  
  try {
    const writes = [...writeQueue];
    writeQueue.length = 0;
    
    // 배치로 DB에 쓰기 (부분 실패 처리)
    const results = await Promise.allSettled(
      writes.map(async (write) => {
        try {
          await write.writeFn();
        } catch (error) {
          // 재시도 가능한 경우 큐에 다시 추가
          if (write.retries < MAX_RETRIES) {
            writeQueue.push({
              ...write,
              retries: write.retries + 1
            });
          } else {
            // 최대 재시도 횟수 초과 - 에러 로깅
            console.error(`Failed to write after ${MAX_RETRIES} retries:`, {
              key: write.key,
              error
            });
          }
          throw error;
        }
      })
    );
    
    // 실패한 작업 로깅
    const failures = results.filter(r => r.status === 'rejected');
    if (failures.length > 0) {
      console.warn(`${failures.length} write operations failed`);
    }
  } finally {
    isFlushing = false;
  }
}

/**
 * 플러시 타이머를 시작합니다.
 */
function startFlushTimer(): void {
  if (flushTimer) return;
  
  flushTimer = setInterval(() => {
    flushQueue().catch((error) => {
      console.error('Error flushing write queue:', error);
    });
  }, FLUSH_INTERVAL_MS);
  
  // 프로세스 종료 시 큐 플러시
  process.on('SIGTERM', async () => {
    if (flushTimer) {
      clearInterval(flushTimer);
      flushTimer = null;
    }
    await flushQueue();
  });
  
  process.on('SIGINT', async () => {
    if (flushTimer) {
      clearInterval(flushTimer);
      flushTimer = null;
    }
    await flushQueue();
  });
}

/**
 * Write-Behind 패턴으로 캐시와 DB에 쓰기합니다.
 */
export async function writeBehind<T>(
  cacheKey: string,
  value: T,
  writeFn: () => Promise<void>,
  ttl?: number
): Promise<void> {
  // 큐 크기 제한 확인
  if (writeQueue.length >= MAX_QUEUE_SIZE) {
    // 큐가 가득 찬 경우 즉시 플러시 시도
    await flushQueue();
    
    if (writeQueue.length >= MAX_QUEUE_SIZE) {
      throw new CacheError('Write queue is full. Please retry later.');
    }
  }
  
  // 1. 캐시에 즉시 업데이트
  try {
    await cache.set(cacheKey, value, ttl);
  } catch (error) {
    throw new CacheError(`Failed to set cache for key: ${cacheKey}`, error);
  }
  
  // 2. DB 쓰기를 큐에 추가 (비동기)
  writeQueue.push({
    key: cacheKey,
    value,
    writeFn,
    retries: 0,
    timestamp: Date.now()
  });
  
  // 3. 플러시 타이머 시작 (아직 시작되지 않았다면)
  startFlushTimer();
}

/**
 * 큐에 남은 모든 작업을 즉시 플러시합니다.
 */
export async function flushAll(): Promise<void> {
  if (flushTimer) {
    clearInterval(flushTimer);
    flushTimer = null;
  }
  await flushQueue();
}
```

> [!TIP]
> 프로덕션 환경에서는 Redis Streams, RabbitMQ, 또는 다른 영속적 메시지 큐를 사용하여 Write-Behind 패턴을 구현하는 것이 좋습니다. 이렇게 하면 서버 재시작 시에도 데이터 손실을 방지할 수 있습니다.

```typescript
// src/routes/products/+page.server.ts
import { writeBehind } from '$lib/server/cache/write-behind';
import { cacheKeys } from '$lib/server/cache/keys';
import { cache } from '$lib/server/cache';
import { invalidate } from '$app/navigation';

export const actions = {
  update: async ({ request, params }) => {
    const formData = await request.formData();
    const updates = Object.fromEntries(formData);
    
    // 기존 상품 데이터 조회 (캐시 또는 DB)
    const existingProduct = await cache.get(cacheKeys.products.byId(params.id)) 
      || await db.getProduct(params.id);
    
    // Write-behind: 캐시에만 먼저 업데이트
    const updatedProduct = { ...existingProduct, ...updates };
    
    await writeBehind(
      cacheKeys.products.byId(params.id),
      updatedProduct,
      async () => {
        // 나중에 DB에 쓰기 (비동기)
        await db.updateProduct(params.id, updates);
      },
      300
    );
    
    // 사용자는 즉시 업데이트된 데이터를 볼 수 있음
    await invalidate('app:products');
    
    return { success: true, product: updatedProduct };
  }
};
```

---

## 트레이드오프 분석

| 패턴 | 장점 | 단점 | 사용 사례 |
|------|------|------|-----------|
| **Write-Through** | 데이터 일관성 보장, 안전함 | 쓰기 성능 느림 | 금융 데이터, 중요한 트랜잭션 |
| **Write-Behind** | 쓰기 성능 빠름, 사용자 경험 좋음 | 데이터 손실 위험, 복잡함 | 조회수, 좋아요, 로그 데이터 |

---

## FastAPI에서 Write-Through

```python
# backend/apps/api/src/app/cache/write_through.py
from typing import Callable, TypeVar, Awaitable
from app.cache import get_cache

T = TypeVar('T')

async def write_through(
    cache_key: str,
    write_fn: Callable[[], Awaitable[T]],
    ttl: int | None = None
) -> T:
    cache = await get_cache()
    
    # 1. 데이터베이스에 쓰기
    result = await write_fn()
    
    # 2. 캐시에 즉시 업데이트
    await cache.set(cache_key, result, ttl)
    
    return result
```

```python
# backend/apps/api/src/app/features/products/router.py
from app.cache.write_through import write_through

@router.put("/products/{product_id}")
async def update_product(
    product_id: int,
    updates: ProductUpdate,
    cache: CacheAdapter = Depends(get_cache)
):
    cache_key = f"products:{product_id}"
    
    # Write-through 패턴
    updated_product = await write_through(
        cache_key,
        lambda: db.update_product(product_id, updates),
        ttl=300
    )
    
    return updated_product
```

