# TanStack Query (React Query v5+) — React Native Patterns

## QueryClient Setup

```typescript
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,   // 5 minutes
      gcTime: 1000 * 60 * 30,     // 30 minutes
      retry: 2,
      refetchOnWindowFocus: false, // not useful in RN
      refetchOnReconnect: true,
    },
    mutations: {
      retry: 1,
    },
  },
});
```

## Custom Hook Pattern (Queries)

Always encapsulate queries in custom hooks. Never call `useQuery` directly in components.

```typescript
// src/features/user/hooks/useUserQuery.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/lib/apiClient';
import type { User } from '../types';

interface UseUserQueryOptions {
  enabled?: boolean;
}

export const userKeys = {
  all: ['users'] as const,
  detail: (id: string) => [...userKeys.all, id] as const,
  list: (filters: UserFilters) => [...userKeys.all, 'list', filters] as const,
};

export function useUserQuery(userId: string, options?: UseUserQueryOptions) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: async (): Promise<User> => {
      const response = await apiClient.get<User>(`/users/${userId}`);
      return response.data;
    },
    enabled: options?.enabled ?? true,
    staleTime: 1000 * 60 * 10, // 10 min — user data rarely changes
  });
}
```

## Custom Hook Pattern (Mutations)

```typescript
// src/features/auth/hooks/useLoginMutation.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { apiClient } from '@/lib/apiClient';
import { useAuthStore } from '../store/authStore';
import type { LoginPayload, AuthResponse } from '../types';

export function useLoginMutation() {
  const queryClient = useQueryClient();
  const setToken = useAuthStore((state) => state.setToken);

  return useMutation({
    mutationFn: async (payload: LoginPayload): Promise<AuthResponse> => {
      const response = await apiClient.post<AuthResponse>('/auth/login', payload);
      return response.data;
    },
    onSuccess: (data) => {
      setToken(data.accessToken);
      // Prefetch user profile after login
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
    onError: (error) => {
      // Let the component handle error display
      console.error('Login failed:', error);
    },
  });
}
```

## Query Key Factory Pattern

Centralize query keys per feature to avoid typos and enable targeted invalidation:

```typescript
export const productKeys = {
  all: ['products'] as const,
  lists: () => [...productKeys.all, 'list'] as const,
  list: (filters: ProductFilters) => [...productKeys.lists(), filters] as const,
  details: () => [...productKeys.all, 'detail'] as const,
  detail: (id: string) => [...productKeys.details(), id] as const,
};
```

## Infinite Queries (Pagination)

```typescript
export function useProductsInfiniteQuery(filters: ProductFilters) {
  return useInfiniteQuery({
    queryKey: productKeys.list(filters),
    queryFn: async ({ pageParam }): Promise<PaginatedResponse<Product>> => {
      const response = await apiClient.get('/products', {
        params: { ...filters, cursor: pageParam },
      });
      return response.data;
    },
    initialPageParam: undefined as string | undefined,
    getNextPageParam: (lastPage) => lastPage.nextCursor ?? undefined,
  });
}
```

## Optimistic Updates

```typescript
export function useToggleFavoriteMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (productId: string) =>
      apiClient.post(`/products/${productId}/favorite`),
    onMutate: async (productId) => {
      await queryClient.cancelQueries({ queryKey: productKeys.detail(productId) });

      const previous = queryClient.getQueryData<Product>(
        productKeys.detail(productId),
      );

      queryClient.setQueryData<Product>(
        productKeys.detail(productId),
        (old) => old ? { ...old, isFavorite: !old.isFavorite } : old,
      );

      return { previous };
    },
    onError: (_err, productId, context) => {
      if (context?.previous) {
        queryClient.setQueryData(productKeys.detail(productId), context.previous);
      }
    },
    onSettled: (_data, _err, productId) => {
      queryClient.invalidateQueries({ queryKey: productKeys.detail(productId) });
    },
  });
}
```

## Select for Re-render Optimization

Use `select` to derive data and avoid re-renders when irrelevant fields change:

```typescript
// Only re-renders when the user's name changes, not the entire user object
export function useUserName(userId: string) {
  return useQuery({
    queryKey: userKeys.detail(userId),
    queryFn: () => fetchUser(userId),
    select: (data) => data.name,
  });
}
```

## Error and Loading Boundaries

Prefer per-query error/loading handling in React Native since `Suspense` support is limited:

```typescript
const { data, isLoading, isError, error, refetch } = useUserQuery(userId);

if (isLoading) return <LoadingSkeleton />;
if (isError) return <ErrorView message={error.message} onRetry={refetch} />;
```
