# Cache Security

사용자별 데이터를 캐싱할 때는 보안을 최우선으로 고려해야 합니다. 다른 사용자의 데이터가 노출되거나 캐시 오염이 발생하지 않도록 주의해야 합니다.

---

## private vs public 캐시

```typescript
// ❌ BAD: 사용자별 데이터를 public으로 캐시
setHeaders({
  'cache-control': 'public, max-age=3600' // CDN에서 다른 사용자에게 노출 가능
});

// ✅ GOOD: 사용자별 데이터는 private
setHeaders({
  'cache-control': 'private, max-age=300' // 브라우저에만 캐시, CDN은 캐시하지 않음
});
```

---

## 사용자 ID 기반 캐시 키

캐시 키에 사용자 ID를 포함하여 사용자별로 분리합니다.

```typescript
// src/lib/server/cache/keys.ts
export const cacheKeys = {
  user: {
    dashboard: (userId: string) => `app:user:${userId}:dashboard`,
    profile: (userId: string) => `app:user:${userId}:profile`,
    settings: (userId: string) => `app:user:${userId}:settings`
  }
};
```

```typescript
// src/routes/dashboard/+page.server.ts
import { cache } from '$lib/server/cache';
import { cacheKeys } from '$lib/server/cache/keys';

export const load: PageServerLoad = async ({ locals, setHeaders }) => {
  const userId = locals.user.id;
  const cacheKey = cacheKeys.user.dashboard(userId);
  
  // 사용자별 캐시 조회
  let dashboardData = await cache.get(cacheKey);
  
  if (!dashboardData) {
    dashboardData = await db.getUserDashboard(userId);
    await cache.set(cacheKey, dashboardData, 300); // 5분 TTL
  }
  
  // HTTP 헤더도 private로 설정
  setHeaders({
    'cache-control': 'private, max-age=300'
  });
  
  return { dashboardData };
};
```

---

## 캐시 오염 방지 (Cache Poisoning)

캐시 키에 사용자 입력을 직접 사용하지 마세요. 항상 검증하고 정규화하세요.

```typescript
// ❌ BAD: 사용자 입력을 직접 캐시 키에 사용
const cacheKey = `products:${userInput}`; // 위험!
// 예: userInput = "../../etc/passwd" → 캐시 키 오염 가능

// ❌ BAD: 민감한 정보를 캐시 키에 포함
const cacheKey = `user:${userId}:token:${authToken}`; // 토큰 노출 위험!

// ✅ GOOD: 입력 검증 및 정규화 (cacheKeys 유틸 사용)
import { cacheKeys } from '$lib/server/cache/keys';
const cacheKey = cacheKeys.products.byCategory(userInput); // 내부에서 정규화됨

// ✅ GOOD: 검증된 값만 사용
const validatedCategory = validateCategory(userInput); // 검증 로직
const cacheKey = cacheKeys.products.byCategory(validatedCategory);
```

### 캐시 키 정규화 함수

```typescript
// src/lib/server/cache/utils.ts
export function normalizeCacheKey(input: string): string {
  if (!input || typeof input !== 'string') {
    throw new Error('Invalid cache key input');
  }
  
  // 길이 제한 (보안 및 성능)
  if (input.length > 200) {
    throw new Error('Cache key too long');
  }
  
  // 허용된 문자만 사용, 특수문자 제거 및 정규화
  const normalized = input
    .replace(/[^a-zA-Z0-9-_]/g, '-') // 특수문자를 하이픈으로
    .replace(/-+/g, '-') // 연속된 하이픈 제거
    .replace(/^-|-$/g, '') // 앞뒤 하이픈 제거
    .toLowerCase()
    .slice(0, 200); // 최대 길이 제한
  
  if (!normalized) {
    throw new Error('Cache key normalization resulted in empty string');
  }
  
  return normalized;
}
```

---

## 캐시 데이터 암호화

민감한 데이터를 캐시할 때는 암호화를 고려하세요.

```typescript
// src/lib/server/cache/encryption.ts
import crypto from 'crypto';

const ENCRYPTION_KEY = process.env.CACHE_ENCRYPTION_KEY!; // 32바이트 키
const ALGORITHM = 'aes-256-gcm';

export function encryptCacheValue(value: string): string {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv(ALGORITHM, Buffer.from(ENCRYPTION_KEY, 'hex'), iv);
  
  let encrypted = cipher.update(value, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  // IV + AuthTag + 암호화된 데이터
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

export function decryptCacheValue(encrypted: string): string {
  const [ivHex, authTagHex, encryptedData] = encrypted.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  
  const decipher = crypto.createDecipheriv(
    ALGORITHM,
    Buffer.from(ENCRYPTION_KEY, 'hex'),
    iv
  );
  decipher.setAuthTag(authTag);
  
  let decrypted = decipher.update(encryptedData, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  
  return decrypted;
}
```

> [!WARNING]
> 암호화는 성능 오버헤드가 있습니다. 모든 데이터를 암호화할 필요는 없으며, 민감한 정보(개인정보, 결제 정보 등)에만 적용하세요.

---

## Vary 헤더 활용

`Vary` 헤더를 사용하여 요청 헤더에 따라 다른 캐시를 사용하도록 지정할 수 있습니다.

```typescript
// src/routes/products/+page.server.ts
export const load: PageServerLoad = async ({ setHeaders, request }) => {
  const products = await db.getProducts();
  
  // Authorization 헤더에 따라 다른 캐시 사용
  setHeaders({
    'cache-control': 'private, max-age=300',
    'vary': 'Authorization' // 인증 상태에 따라 다른 캐시
  });
  
  return { products };
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

