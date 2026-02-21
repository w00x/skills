# Zustand — React Native Patterns

## Basic Store

```typescript
// src/features/auth/store/authStore.ts
import { create } from 'zustand';

interface AuthState {
  token: string | null;
  isAuthenticated: boolean;
  setToken: (token: string) => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>((set) => ({
  token: null,
  isAuthenticated: false,
  setToken: (token) => set({ token, isAuthenticated: true }),
  logout: () => set({ token: null, isAuthenticated: false }),
}));
```

## Slices Pattern (Complex Stores)

For apps with multiple concerns, split the store into slices to keep each file manageable.

```typescript
// src/store/slices/uiSlice.ts
import type { StateCreator } from 'zustand';
import type { AppStore } from '../appStore';

export interface UISlice {
  theme: 'light' | 'dark';
  isDrawerOpen: boolean;
  toggleTheme: () => void;
  setDrawerOpen: (open: boolean) => void;
}

export const createUISlice: StateCreator<AppStore, [], [], UISlice> = (set) => ({
  theme: 'light',
  isDrawerOpen: false,
  toggleTheme: () =>
    set((state) => ({ theme: state.theme === 'light' ? 'dark' : 'light' })),
  setDrawerOpen: (open) => set({ isDrawerOpen: open }),
});
```

```typescript
// src/store/slices/onboardingSlice.ts
import type { StateCreator } from 'zustand';
import type { AppStore } from '../appStore';

export interface OnboardingSlice {
  currentStep: number;
  isCompleted: boolean;
  nextStep: () => void;
  completeOnboarding: () => void;
  resetOnboarding: () => void;
}

export const createOnboardingSlice: StateCreator<AppStore, [], [], OnboardingSlice> = (set) => ({
  currentStep: 0,
  isCompleted: false,
  nextStep: () => set((state) => ({ currentStep: state.currentStep + 1 })),
  completeOnboarding: () => set({ isCompleted: true }),
  resetOnboarding: () => set({ currentStep: 0, isCompleted: false }),
});
```

```typescript
// src/store/appStore.ts
import { create } from 'zustand';
import { createUISlice, type UISlice } from './slices/uiSlice';
import { createOnboardingSlice, type OnboardingSlice } from './slices/onboardingSlice';

export type AppStore = UISlice & OnboardingSlice;

export const useAppStore = create<AppStore>()((...args) => ({
  ...createUISlice(...args),
  ...createOnboardingSlice(...args),
}));
```

## Selectors & useShallow

**Always use selectors** to pick only the state your component needs. Use `useShallow` when selecting multiple properties to avoid re-renders from shallow-equal reference changes.

```typescript
import { useShallow } from 'zustand/react/shallow';

// ❌ BAD — re-renders on ANY state change
const state = useAppStore();

// ❌ BAD — re-renders when any slice property changes (new object ref every time)
const { theme, isDrawerOpen } = useAppStore((s) => ({
  theme: s.theme,
  isDrawerOpen: s.isDrawerOpen,
}));

// ✅ GOOD — single primitive selector, no wrapper needed
const theme = useAppStore((s) => s.theme);

// ✅ GOOD — multiple properties with useShallow
const { theme, isDrawerOpen } = useAppStore(
  useShallow((s) => ({ theme: s.theme, isDrawerOpen: s.isDrawerOpen })),
);
```

## Persistence with MMKV

Use `zustand/middleware` + `react-native-mmkv` for high-performance persistence:

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';
import { MMKV } from 'react-native-mmkv';

const storage = new MMKV();

const mmkvStorage = {
  getItem: (name: string) => storage.getString(name) ?? null,
  setItem: (name: string, value: string) => storage.set(name, value),
  removeItem: (name: string) => storage.delete(name),
};

interface SettingsState {
  locale: string;
  notificationsEnabled: boolean;
  setLocale: (locale: string) => void;
  toggleNotifications: () => void;
}

export const useSettingsStore = create<SettingsState>()(
  persist(
    (set) => ({
      locale: 'en',
      notificationsEnabled: true,
      setLocale: (locale) => set({ locale }),
      toggleNotifications: () =>
        set((s) => ({ notificationsEnabled: !s.notificationsEnabled })),
    }),
    {
      name: 'settings-storage',
      storage: createJSONStorage(() => mmkvStorage),
    },
  ),
);
```

## Secure Persistence with expo-secure-store

For sensitive data (tokens, secrets), use `expo-secure-store` instead of MMKV:

```typescript
import * as SecureStore from 'expo-secure-store';
import { create } from 'zustand';

interface AuthState {
  token: string | null;
  isAuthenticated: boolean;
  setToken: (token: string) => void;
  logout: () => void;
  hydrate: () => Promise<void>;
}

export const useAuthStore = create<AuthState>((set) => ({
  token: null,
  isAuthenticated: false,
  setToken: async (token) => {
    await SecureStore.setItemAsync('auth_token', token);
    set({ token, isAuthenticated: true });
  },
  logout: async () => {
    await SecureStore.deleteItemAsync('auth_token');
    set({ token: null, isAuthenticated: false });
  },
  hydrate: async () => {
    const token = await SecureStore.getItemAsync('auth_token');
    if (token) set({ token, isAuthenticated: true });
  },
}));
```

> Use MMKV for non-sensitive data (preferences, UI state). Use `expo-secure-store` for tokens and credentials.

## Accessing Store Outside React

```typescript
// Useful in navigation guards, interceptors, etc.
const token = useAuthStore.getState().token;
useAuthStore.getState().logout();
```

## Computed / Derived State

Derive values inside selectors to avoid storing redundant state:

```typescript
// Derived selector (not stored in the store)
const isOnboardingActive = useAppStore(
  (s) => !s.isCompleted && s.currentStep > 0,
);
```

## When to Use Zustand vs TanStack Query

| Zustand | TanStack Query |
|---------|---------------|
| Auth tokens, session info | API responses (users, products, etc.) |
| UI state (theme, drawer, modals) | Paginated / infinite lists |
| Onboarding progress | Server-driven feature flags |
| User preferences (locale, units) | Any data with a remote source of truth |
| Navigation state flags | Search results |
