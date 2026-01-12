# Access Control Patterns

## Production-Level RBAC

### Pattern 1: Route-Level Control (Layouts)

Check permissions in `layout.tsx` to protect entire route groups.

```typescript
// app/(app)/admin/layout.tsx
import { redirect } from 'next/navigation';
import { getUser, getUserRole } from '@/features/auth/server';

export default async function AdminLayout({ children }: { children: React.ReactNode }) {
  const user = await getUser();
  if (!user) redirect('/login');

  const role = await getUserRole(user.id);
  if (role !== 'admin') redirect('/dashboard?error=unauthorized');

  return <>{children}</>;
}
```

**Note:** For API routes or when you have JWT tokens directly (e.g., from Authorization header), consider using `getClaims()` instead of `getUser()` for better performance. `getClaims()` verifies JWT via JWKS without Auth server round-trip.

### Pattern 2: Component-Level Conditional Rendering

Conditionally render content based on roles within Server Components.

```typescript
export async function AdminPanel() {
  const user = await getUser();
  const role = await getUserRole(user.id);
  
  if (role === 'admin') return <AdminContent />;
  if (role === 'moderator') return <ModeratorContent />;
  
  return <UnauthorizedMessage />;
}
```

**Note:** If you only need user ID from JWT claims (not full user object), you can use `getClaims()` for faster verification. However, `getUser()` is recommended here as it ensures session validity.

### Pattern 3: Feature-Level Access Check (Recommended)

Encapsulate permission logic within the Feature's Server Component Container.

```typescript
// features/analytics/components/AnalyticsContent.tsx
import { checkPermission } from '@/features/auth/server';

export async function AnalyticsContent() {
  const user = await getUser();
  const hasAccess = await checkPermission(user.id, 'analytics:read');
  
  if (!hasAccess) return <UnauthorizedView />;

  // Only fetch data if authorized
  const data = await fetchAnalytics(user.id);
  return <AnalyticsView data={data} />;
}
```

**Note:** In Server Components with cookie-based sessions, `getUser()` is appropriate. For API routes or FastAPI endpoints receiving Bearer tokens, use `getClaims()` for JWT verification instead.

### Pattern 4: Dynamic Route Protection (Parallel Routes)

Use Next.js Parallel Routes (`@admin`, `@user`) to conditionally render slots based on roles.

### Permission Service Utility

```typescript
// features/auth/server.ts
import 'server-only';
import { cacheLife, cacheTag } from 'next/cache';

export async function checkPermission(userId: string, permission: string): Promise<boolean> {
  'use cache';
  cacheLife('minutes');
  cacheTag(`user-permission-${userId}-${permission}`);
  
  const supabase = await createClient();
  const { data } = await supabase
    .from('user_permissions')
    .select('*')
    .eq('user_id', userId)
    .eq('permission', permission)
    .single();
    
  return !!data;
}
```
