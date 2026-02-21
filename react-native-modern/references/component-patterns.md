# Expo + React Native Component & Performance Patterns

## Functional Component Template

```typescript
import React, { memo, useCallback } from 'react';
import { View, Text, Pressable, StyleSheet } from 'react-native';

interface ProductCardProps {
  id: string;
  name: string;
  price: number;
  onPress: (id: string) => void;
}

export const ProductCard = memo<ProductCardProps>(({ id, name, price, onPress }) => {
  const handlePress = useCallback(() => onPress(id), [id, onPress]);

  return (
    <Pressable onPress={handlePress} style={styles.card}>
      <Text style={styles.name}>{name}</Text>
      <Text style={styles.price}>${price.toFixed(2)}</Text>
    </Pressable>
  );
});

ProductCard.displayName = 'ProductCard';

const styles = StyleSheet.create({
  card: {
    padding: 16,
    borderRadius: 12,
    backgroundColor: '#fff',
    marginBottom: 8,
    shadowColor: '#000',
    shadowOpacity: 0.05,
    shadowRadius: 4,
    elevation: 2,
  },
  name: { fontSize: 16, fontWeight: '600' },
  price: { fontSize: 14, color: '#666', marginTop: 4 },
});
```

## FlashList over FlatList

Always prefer `@shopify/flash-list` for lists — it recycles views for better performance:

```typescript
import { FlashList } from '@shopify/flash-list';

interface ProductListProps {
  products: Product[];
  onProductPress: (id: string) => void;
}

export function ProductList({ products, onProductPress }: ProductListProps) {
  const renderItem = useCallback(
    ({ item }: { item: Product }) => (
      <ProductCard
        id={item.id}
        name={item.name}
        price={item.price}
        onPress={onProductPress}
      />
    ),
    [onProductPress],
  );

  return (
    <FlashList
      data={products}
      renderItem={renderItem}
      estimatedItemSize={80}
      keyExtractor={(item) => item.id}
    />
  );
}
```

## Full Feature Screen Pattern

Combines TanStack Query + Zustand + FlashList:

```typescript
// app/(tabs)/index.tsx  OR  src/features/products/screens/ProductListScreen.tsx
import React, { useCallback } from 'react';
import { View, ActivityIndicator, StyleSheet } from 'react-native';
import { FlashList } from '@shopify/flash-list';
import { useProductsInfiniteQuery } from '../hooks/useProductsInfiniteQuery';
import { useAppStore } from '@/store/appStore';
import { ProductCard } from '../components/ProductCard';
import { ErrorView } from '@/components/ui/ErrorView';
import type { Product } from '../types';

export function ProductListScreen() {
  const theme = useAppStore((s) => s.theme);
  const {
    data,
    isLoading,
    isError,
    error,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    refetch,
  } = useProductsInfiniteQuery({ category: 'all' });

  const products = data?.pages.flatMap((page) => page.items) ?? [];

  const handleProductPress = useCallback((id: string) => {
    router.push(`/product/${id}`);
  }, []);

  const renderItem = useCallback(
    ({ item }: { item: Product }) => (
      <ProductCard id={item.id} name={item.name} price={item.price} onPress={handleProductPress} />
    ),
    [handleProductPress],
  );

  const handleEndReached = useCallback(() => {
    if (hasNextPage && !isFetchingNextPage) fetchNextPage();
  }, [hasNextPage, isFetchingNextPage, fetchNextPage]);

  if (isLoading) return <ActivityIndicator style={styles.center} />;
  if (isError) return <ErrorView message={error.message} onRetry={refetch} />;

  return (
    <View style={[styles.container, theme === 'dark' && styles.dark]}>
      <FlashList
        data={products}
        renderItem={renderItem}
        estimatedItemSize={80}
        keyExtractor={(item) => item.id}
        onEndReached={handleEndReached}
        onEndReachedThreshold={0.5}
        ListFooterComponent={isFetchingNextPage ? <ActivityIndicator /> : null}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, backgroundColor: '#f9f9f9' },
  dark: { backgroundColor: '#1a1a1a' },
  center: { flex: 1, justifyContent: 'center', alignItems: 'center' },
});
```

## Navigation with Expo Router

Expo Router uses file-based routing — no manual stack/param definitions needed:

```typescript
// app/product/[id].tsx — dynamic route
import { useLocalSearchParams } from 'expo-router';

export default function ProductDetailScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const { data, isLoading } = useProductQuery(id);
  // ...
}
```

```typescript
// Programmatic navigation
import { router } from 'expo-router';

router.push('/product/abc-123');
router.replace('/(auth)/login');
router.back();
```

## API Client Setup

```typescript
// src/lib/apiClient.ts
import axios from 'axios';
import { useAuthStore } from '@/features/auth/store/authStore';

export const apiClient = axios.create({
  baseURL: 'https://api.example.com/v1',
  timeout: 15_000,
  headers: { 'Content-Type': 'application/json' },
});

// Inject auth token automatically
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token;
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Handle 401 globally
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
    }
    return Promise.reject(error);
  },
);
```

## Performance Checklist

1. **Memoize components** that receive complex props or appear in lists → `React.memo`
2. **Stabilize callbacks** passed to children → `useCallback`
3. **Use selectors** with Zustand → single primitive or `useShallow`
4. **Use `select`** with TanStack Query → derive only needed fields
5. **Use FlashList** → always over FlatList for scrollable lists
6. **Avoid inline objects/arrays** in JSX props → extract to `useMemo` or constants
7. **Extract StyleSheet** → always use `StyleSheet.create` outside the component
