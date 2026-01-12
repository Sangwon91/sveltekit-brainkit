# Data Serialization & Boundaries

## Server to Client Boundary
Passing data from Server Component -> Client Component requires **Serialization**.
-   **Supported**: JSON-serializable data (Strings, Numbers, Objects, Arrays).
-   **Unsupported**: Functions, Classes, Date objects (converted to string), Recursive structures.

## Best Practices
1.  **Keep it Thin**: Don't pass massive objects if only a few fields are needed. Map data on the server first.
2.  **DTOs**: Use simple interfaces for props to ensure type safety across the boundary.

### Props Pattern
```typescript
// Server Component
const data = await fetchUser()
return <UserProfile user={data} /> // Serialized here
```

### Context Pattern
For deep trees, fetch on server, pass to a Client Context Provider.
```typescript
// layout.tsx
const user = await fetchUser()
return (
  <UserProvider initialUser={user}>
    {children}
  </UserProvider>
)
```
