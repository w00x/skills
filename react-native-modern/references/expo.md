# Expo — Setup, Router & SDK Patterns

## Project Initialization

```bash
# Create new Expo project with TypeScript template
npx -y create-expo-app@latest ./ --template tabs

# Install core dependencies
npx expo install @tanstack/react-query zustand axios
npx expo install @shopify/flash-list expo-image expo-secure-store expo-haptics
npx expo install react-native-mmkv
```

## Root Layout with Providers

```typescript
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { useFonts } from 'expo-font';
import * as SplashScreen from 'expo-splash-screen';
import { queryClient } from '@/lib/queryClient';

SplashScreen.preventAutoHideAsync();

export default function RootLayout() {
  const [fontsLoaded] = useFonts({
    Inter: require('@/assets/fonts/Inter.ttf'),
  });

  useEffect(() => {
    if (fontsLoaded) SplashScreen.hideAsync();
  }, [fontsLoaded]);

  if (!fontsLoaded) return null;

  return (
    <QueryClientProvider client={queryClient}>
      <Stack screenOptions={{ headerShown: false }}>
        <Stack.Screen name="(tabs)" />
        <Stack.Screen name="(auth)" />
        <Stack.Screen name="product/[id]" options={{ presentation: 'modal' }} />
      </Stack>
    </QueryClientProvider>
  );
}
```

## Expo Router Navigation Patterns

### Tab Layout

```typescript
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#6366f1' }}>
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: 'Search',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

### Type-Safe Navigation

```typescript
// With Expo Router, use typed params via useLocalSearchParams
import { useLocalSearchParams, router, Link } from 'expo-router';

// In a dynamic route (app/product/[id].tsx):
export default function ProductDetailScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  // ...
}

// Programmatic navigation:
router.push('/product/abc-123');
router.replace('/(auth)/login');
router.back();

// Declarative navigation:
<Link href="/product/abc-123" asChild>
  <Pressable><Text>View Product</Text></Pressable>
</Link>
```

### Auth Guard Pattern

```typescript
// app/(auth)/_layout.tsx
import { Redirect, Stack } from 'expo-router';
import { useAuthStore } from '@/features/auth/store/authStore';

export default function AuthLayout() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

  // Redirect to home if already logged in
  if (isAuthenticated) return <Redirect href="/(tabs)" />;

  return <Stack screenOptions={{ headerShown: false }} />;
}
```

```typescript
// app/(tabs)/_layout.tsx — protect authenticated routes
import { Redirect } from 'expo-router';
import { useAuthStore } from '@/features/auth/store/authStore';

export default function TabLayout() {
  const isAuthenticated = useAuthStore((s) => s.isAuthenticated);

  if (!isAuthenticated) return <Redirect href="/(auth)/login" />;

  // ... tab config
}
```

## Expo SDK Module Patterns

### expo-secure-store (Tokens)

```typescript
import * as SecureStore from 'expo-secure-store';

const TOKEN_KEY = 'auth_token';

export async function saveToken(token: string): Promise<void> {
  await SecureStore.setItemAsync(TOKEN_KEY, token);
}

export async function getToken(): Promise<string | null> {
  return SecureStore.getItemAsync(TOKEN_KEY);
}

export async function deleteToken(): Promise<void> {
  await SecureStore.deleteItemAsync(TOKEN_KEY);
}
```

### expo-image (Optimized Images)

```typescript
import { Image } from 'expo-image';

// Prefer expo-image over RN Image — supports caching, blurhash, transitions
<Image
  source={{ uri: product.imageUrl }}
  placeholder={{ blurhash: product.blurhash }}
  contentFit="cover"
  transition={200}
  style={{ width: '100%', height: 200, borderRadius: 12 }}
/>
```

### expo-haptics

```typescript
import * as Haptics from 'expo-haptics';

// Light feedback on button press
const handlePress = useCallback(() => {
  Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
  onPress(id);
}, [id, onPress]);
```

## Environment Configuration

```typescript
// app.config.ts
import type { ExpoConfig, ConfigContext } from 'expo/config';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: 'MyApp',
  slug: 'my-app',
  extra: {
    apiUrl: process.env.API_URL ?? 'https://api.example.com/v1',
    eas: {
      projectId: process.env.EAS_PROJECT_ID,
    },
  },
});
```

```typescript
// Access in code:
import Constants from 'expo-constants';

const API_URL = Constants.expoConfig?.extra?.apiUrl as string;
```

## EAS Build & Submit

```bash
# Install EAS CLI
npm install -g eas-cli

# Configure EAS
eas build:configure

# Development build (includes dev client)
eas build --platform ios --profile development
eas build --platform android --profile development

# Production build
eas build --platform all --profile production

# Submit to stores
eas submit --platform ios
eas submit --platform android
```

### eas.json Example

```json
{
  "cli": { "version": ">= 12.0.0" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  },
  "submit": {
    "production": {}
  }
}
```
