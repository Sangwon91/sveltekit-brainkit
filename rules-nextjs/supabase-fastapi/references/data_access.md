# Data Access Patterns

## Decision Tree

```
Need Data?
│
├─ Can query Supabase directly?
│  ├─ YES → Use Next.js directly (RLS protected)
│  │         └─ queries.ts with 'use cache'
│  │
│  └─ NO → Need complex processing or external integration?
│           ├─ YES → Next.js → FastAPI
│           │        └─ FastAPI → Supabase (Service Role + Manual Check)
│           │
│           └─ NO → Stick to Next.js
```

## Pattern 1: Direct Access from Next.js (Recommended)

-   **Scenario**: Standard CRUD ops.
-   **Client**: `createClient()` (uses Publishable Key + User JWT, legacy: Anon Key).
-   **Security**: RLS is automatically applied.
-   **Caching**: fully compatible with `'use cache'`.

```typescript
// features/users/queries.ts
import 'server-only';
import { createClient } from '@/lib/supabase/server';

export async function getUserProfile(userId: string) {
  'use cache';
  cacheLife('minutes');
  cacheTag(`user-${userId}`);
  
  const supabase = await createClient();
  const { data, error } = await supabase
    .from('profiles')
    .select('*')
    .eq('id', userId)
    .single();
    
  if (error) throw error;
  return data;
}
```

## Pattern 2: Next.js → FastAPI → Supabase (Hybrid)

-   **Scenario**: Batch jobs, AI processing, heavy logic.
-   **Flow**:
    1.  Next.js Server Component calls FastAPI, passing JWT in header.
    2.  FastAPI Middleware validates JWT.
    3.  FastAPI uses **Secret Key Client** to access Supabase (legacy: Service Role Client).
    4.  **CRITICAL**: Secret Key bypasses RLS. You MUST manually filter by `user_id`.

```typescript
// features/analytics/queries.ts
export async function getAnalytics() {
  const supabase = await createClient();
  const { data: { session } } = await supabase.auth.getSession();
  
  // Pass user's JWT
  const response = await getFromBackend(
    '/api/v1/analytics',
    {},
    session?.access_token
  );
  return response.json();
}
```

```python
# FastAPI Route
@router.get("/analytics")
async def get_analytics(user_id: str = Depends(get_current_user_id)):
    supabase = get_secret_client()  # legacy: get_service_role_client()
    
    # MANUAL RLS: Must filter by user_id explicitly
    return supabase.table('analytics').select('*').eq('user_id', user_id).execute()
```
