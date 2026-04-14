---
name: react-practices
description: React component and hook conventions — PascalCase file organization, container/render separation, hook patterns, SCSS module styling with mobile-first responsive design, and domain structure. Extends ts-patterns with React-specific rules. Use this skill whenever writing or reviewing React components, creating hooks, writing component styles, setting up React project structure, or discussing React best practices. Also use when the user mentions JSX, TSX, React components, hooks, SCSS, CSS Modules, responsive design, or any React UI work.
---

## Overview

This skill extends **ts-patterns** with React-specific conventions. Everything in ts-patterns still applies (functional architecture, naming discipline, barrel exports, type conventions, test philosophy, JSDoc standards) unless explicitly overridden below. When a rule here conflicts with ts-patterns, this skill wins.

The key differences from ts-patterns:

- **PascalCase** for component and domain file/folder names (instead of kebab-case)
- **Container/Render separation** for components
- **Hook conventions** with `use` prefix (camelCase)
- **SCSS Modules** with snake_case class names, flex-first layout, mobile-first responsive design
- File suffixes (`.constants.ts`, `.types.ts`, etc.) remain the same pattern, just with PascalCase domain prefixes

---

## File and Folder Naming

### PascalCase replaces kebab-case

In React projects, all domain folders and files use **PascalCase** instead of the kebab-case convention from ts-patterns. This aligns filenames with the component and type names they export, eliminating the mental mapping between `user-profile.tsx` and `UserProfile`.

| ts-patterns | react-practices |
|---|---|
| `user-profile/` | `UserProfile/` |
| `user-profile.ts` | `UserProfile.ts` |
| `user-profile.types.ts` | `UserProfile.types.ts` |
| `user-profile.constants.ts` | `UserProfile.constants.ts` |
| `user-profile.utils.ts` | `UserProfile.utils.ts` |
| `user-profile.test.ts` | `UserProfile.test.ts` |

### Hooks use camelCase with `use` prefix

Hook files and folders follow standard React convention -- camelCase with the `use` prefix:

| File/Folder | Example |
|---|---|
| Hook folder | `useAuth/` |
| Hook file | `useAuth.ts` |
| Hook context | `useAuth.context.tsx` |
| Hook types | `useAuth.types.ts` |
| Hook constants | `useAuth.constants.ts` |
| Hook tests | `useAuth.test.ts` |

### Suffixes stay the same

The file suffixes from ts-patterns are unchanged. Only the domain prefix casing changes. Note that `test.utils.ts` and `index.ts` have no domain prefix, so they stay lowercase:

| File | Purpose | Exported from barrel? |
|---|---|---|
| `index.ts` | Barrel -- only re-exports | IS the barrel |
| `{Domain}.tsx` | Component (container + render) | Yes |
| `{domain}.context.tsx` | Context object and provider (hook domains only) | Yes (provider) |
| `{Domain}.module.scss` | Scoped component styles (CSS Modules) | No (internal) |
| `{Domain}.types.ts` | All types for this domain | Yes |
| `{Domain}.constants.ts` | Named constants, default value objects | No (internal) |
| `{Domain}.utils.ts` | Pure stateless helpers | No (internal) |
| `{Domain}.test.tsx` | Co-located unit tests | No |
| `{Domain}.storybook.tsx` | Storybook stories (if project uses Storybook) | No |
| `test.utils.ts` | Shared test fixtures for the module | No |

---

## Component Pattern: Container/Render Separation

Every component is split into two functions within the same file: a **container** that owns logic, and a **render** function that is pure.

### Why separate them?

The render function is a pure function of its props -- given the same inputs, it always returns the same JSX. This makes it trivially testable, easy to reason about, and straightforward to use in Storybook stories or snapshot tests in isolation. The container handles the messy parts: state, effects, data fetching, event handler wiring. By keeping these concerns in separate functions, you can test rendering without mocking hooks and test logic without rendering DOM.

### Structure

The container comes first in the file, the render function comes last. Each interface sits directly above the function it belongs to, keeping the type contract and its implementation together.

```tsx
export interface UserProfileProps {
  /** Unique identifier for the user to display. */
  readonly userId: string;
  /** Whether to show the online status indicator. */
  readonly showStatus?: boolean;
}

export function UserProfile(props: UserProfileProps): React.ReactElement {
  const { userId, showStatus = true } = props;
  const { user, isOnline } = useAuth(userId);
  const navigate = useNavigate();

  const handleLogout = React.useCallback((): void => {
    logout();
    navigate("/login");
  }, [navigate]);

  return (
    <UserProfileRender
      displayName={user.displayName}
      avatarUrl={user.avatarUrl}
      isOnline={showStatus && isOnline}
      onLogout={handleLogout}
    />
  );
}

export interface UserProfileRenderProps {
  /** Display name shown next to the avatar. */
  readonly displayName: string;
  /** URL for the user's avatar image. */
  readonly avatarUrl: string;
  /** Whether the user is currently online. */
  readonly isOnline: boolean;
  /** Callback invoked when the user clicks log out. */
  readonly onLogout: () => void;
}

export function UserProfileRender(props: UserProfileRenderProps): React.ReactElement {
  const { displayName, avatarUrl, isOnline, onLogout } = props;

  return (
    <div>
      <img src={avatarUrl} alt={displayName} />
      <span>{displayName}</span>
      {isOnline && <span>Online</span>}
      <button onClick={onLogout}>Log out</button>
    </div>
  );
}
```

### Rules

1. **The container is the primary public API.** `UserProfile` is what consumers import and use in their component trees. It is exported from the barrel.

2. **The render function is also exported.** `UserProfileRender` is exported from the barrel alongside the container. This enables visual regression testing (e.g., Storybook stories) and isolated rendering tests where you want to control props directly without wiring up the container's dependencies.

3. **Container first, render last.** The container comes first in the file because it's the primary API consumers care about. The render function follows below it. Each interface sits directly above the function it belongs to -- `{Component}Props` above the container, `{Component}RenderProps` above the render function. This keeps the type contract co-located with its implementation rather than grouping all interfaces at the top of the file.

4. **The render function focuses on rendering.** It receives its data through props and returns JSX. It should not trigger application-wide side effects or data mutations. However, rendering-specific concerns are allowed inside the render function: local UI state (e.g., `useState` for a tooltip open/closed toggle), layout effects (`useLayoutEffect` for measuring DOM elements), animations, and derived rendering state from prop changes. The line is clear -- if the state or effect exists purely to control how things look on screen, it belongs in the render function. If it fetches data, mutates application state, or triggers business logic, it belongs in the container.

5. **Both props interfaces are exported and documented.** Define `{Component}Props` for the container and `{Component}RenderProps` for the render function. Both are exported from the types file. Every field in both interfaces gets a single-line `/** */` JSDoc comment explaining its purpose.

6. **Both live in the same file.** Don't split them across files -- the render function is a detail of the component, not a separate module.

7. **Simple components can skip the separation.** If a component has no state, no effects, and no hooks -- just props in, JSX out -- it's already a pure render function. No need to artificially create an empty container around it. Use your judgment: the pattern exists to separate concerns. When there's only one concern, one function is fine.

---

## Hook Conventions

### File organization

Hooks follow the same domain structure but use camelCase with the `use` prefix:

```
useAuth/
  useAuth.ts
  useAuth.types.ts
  useAuth.constants.ts
  useAuth.utils.ts
  useAuth.test.ts
  index.ts
```

### Hook structure

Hooks follow the factory-like pattern from ts-patterns -- they encapsulate state and return a clean public interface. Inner functions that are returned or passed as callbacks must be wrapped in `React.useCallback` so consumers can rely on stable references for memoization and dependency arrays:

```ts
export function useAuth(): AuthState {
  const [currentUser, setCurrentUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const subscription = authService.onAuthChange((user) => {
      setCurrentUser(user);
      setIsLoading(false);
    });

    return () => subscription.unsubscribe();
  }, []);

  const login = React.useCallback(
    async (credentials: LoginCredentials): Promise<void> => {
      // ...
    },
    [],
  );

  const logout = React.useCallback(async (): Promise<void> => {
    // ...
  }, []);

  return {
    currentUser,
    isLoading,
    isAuthenticated: currentUser !== null,
    login,
    logout,
  };
}
```

### Hook naming

- Hook files and folders: `useAuth`, `useFormValidation`, `useLocalStorage`
- Return type interfaces: `AuthState`, `FormValidationState`, `LocalStorageState`
- Internal helpers that don't use hooks go in `useAuth.utils.ts`

---

## Context Conventions

Contexts follow the hook file structure (`use{Domain}`) but split the context creation and provider into a dedicated `{domain}.context.tsx` file, keeping the consumer hook in `use{Domain}.ts`. This gives each file a single clear responsibility: the context file owns the React context object and its provider component, the hook file owns the consumer interface.

### Why null-initialized with a throwing hook?

Initializing context with `null` and throwing in the hook if the value is still `null` gives you a clear, immediate error when a component tries to consume a context that has no provider above it in the tree. Without this, you'd get subtle `undefined` bugs deep in rendering. The throw makes missing providers a loud failure at the exact call site, not a silent `undefined` that surfaces three components down.

### File organization

Context hooks live in the hooks directory alongside regular hooks. The `.context.tsx` file sits next to the hook file:

```
hooks/
  useAuth/
    useAuth.ts
    useAuth.context.tsx
    useAuth.types.ts
    useAuth.test.ts
    index.ts
```

### Structure

Each context domain spans three files:
1. **`use{Domain}.types.ts`** -- the context type, provider props, and any related types
2. **`use{Domain}.context.tsx`** -- the `createContext` call (module-private) and the `{Domain}ContextProvider` component
3. **`use{Domain}.ts`** -- the `use{Domain}` consumer hook that reads the context and throws if the provider is missing

#### Readonly context (pure derived state, no mutations exposed)

When the context provides computed or read-only values that consumers should not mutate:

```ts
// useTheme.types.ts

export interface ThemeContextType {
  /** The active theme name. */
  readonly currentTheme: Theme;
  /** Resolved color tokens for the active theme. */
  readonly colors: ThemeColors;
  /** Resolved spacing tokens for the active theme. */
  readonly spacing: ThemeSpacing;
}

export interface ThemeContextProviderProps {
  /** The theme to apply across all children. */
  readonly themeName: ThemeName;
  /** Child components that can consume the theme. */
  readonly children: React.ReactNode;
}
```

```tsx
// useTheme.context.tsx

import type { ThemeContextType, ThemeContextProviderProps } from "./useTheme.types";

export const ThemeContext = React.createContext<ThemeContextType | null>(null);

export function ThemeContextProvider(props: ThemeContextProviderProps): React.ReactElement {
  const { themeName, children } = props;

  const contextValue = React.useMemo<ThemeContextType>(() => {
    const resolvedTheme = resolveTheme(themeName);
    return {
      currentTheme: resolvedTheme,
      colors: resolvedTheme.colors,
      spacing: resolvedTheme.spacing,
    };
  }, [themeName]);

  return (
    <ThemeContext.Provider value={contextValue}>
      {children}
    </ThemeContext.Provider>
  );
}
```

```ts
// useTheme.ts

import { ThemeContext } from "./useTheme.context";
import type { ThemeContextType } from "./useTheme.types";

export function useTheme(): ThemeContextType {
  const contextValue = React.useContext(ThemeContext);

  if (contextValue === null) {
    throw new Error("useTheme must be used within a ThemeContextProvider");
  }

  return contextValue;
}
```

```ts
// useTheme/index.ts

export { ThemeContextProvider } from "./useTheme.context";
export { useTheme } from "./useTheme";

export type {
  ThemeContextType,
  ThemeContextProviderProps,
} from "./useTheme.types";
```

#### Mutable context (exposes state setters)

When consumers need to both read and update shared state, expose the `useState` tuple directly so the consuming component has full control over state updates:

```ts
// useAuth.types.ts

export interface AuthContextType {
  /** The currently authenticated user, or null if logged out. */
  readonly currentUser: User | null;
  /** Setter for the current user. */
  readonly setCurrentUser: React.Dispatch<React.SetStateAction<User | null>>;
  /** Whether initial auth check is still in progress. */
  readonly isLoading: boolean;
  /** Setter for the loading state. */
  readonly setIsLoading: React.Dispatch<React.SetStateAction<boolean>>;
}

export interface AuthContextProviderProps {
  /** Child components that can consume auth state. */
  readonly children: React.ReactNode;
}
```

```tsx
// useAuth.context.tsx

import type { AuthContextType, AuthContextProviderProps } from "./useAuth.types";

export const AuthContext = React.createContext<AuthContextType | null>(null);

export function AuthContextProvider(props: AuthContextProviderProps): React.ReactElement {
  const { children } = props;
  const [currentUser, setCurrentUser] = React.useState<User | null>(null);
  const [isLoading, setIsLoading] = React.useState(true);

  const contextValue = React.useMemo<AuthContextType>(
    () => ({ currentUser, setCurrentUser, isLoading, setIsLoading }),
    [currentUser, isLoading],
  );

  return (
    <AuthContext.Provider value={contextValue}>
      {children}
    </AuthContext.Provider>
  );
}
```

```ts
// useAuth.ts

import { AuthContext } from "./useAuth.context";
import type { AuthContextType } from "./useAuth.types";

export function useAuth(): AuthContextType {
  const contextValue = React.useContext(AuthContext);

  if (contextValue === null) {
    throw new Error("useAuth must be used within an AuthContextProvider");
  }

  return contextValue;
}
```

```ts
// useAuth/index.ts

export { AuthContextProvider } from "./useAuth.context";
export { useAuth } from "./useAuth";

export type {
  AuthContextType,
  AuthContextProviderProps,
} from "./useAuth.types";
```

### Rules

1. **Always initialize with `null`.** `React.createContext<ContextType | null>(null)` -- never provide a fake default. The null check in the hook is your safety net.

2. **The hook always throws on null.** Every `use{Domain}` hook that wraps a context must check for `null` and throw a descriptive error naming both the hook and the provider. This makes missing providers immediately obvious in development.

3. **Memoize the context value.** Wrap the value object in `React.useMemo` to prevent unnecessary re-renders of all consumers on every provider render. The memo dependencies should include only the values that actually change.

4. **Choose readonly vs. mutable based on the domain.** If the context provides configuration or derived data that consumers just read (theme, locale, feature flags), use a readonly context. If consumers need to update shared state (auth, shopping cart, form state), expose the `useState` setters directly. Don't mix patterns -- if a context is mutable, make that explicit in the type by including the setters.

5. **The raw context object stays in the `.context.tsx` file.** The barrel exports the provider and the consumer hook. Consumers never import the context object directly -- they use the `use{Domain}` hook.

6. **Provider initializes all state.** The provider component is responsible for setting up initial state, running initialization effects, and constructing the context value. Consumers should not need to initialize anything -- they just call the hook and get a ready-to-use value.

---

## Domain Folder Example

```
components/
  UserProfile/
    UserProfile.tsx
    UserProfile.module.scss
    UserProfile.types.ts
    UserProfile.constants.ts
    UserProfile.utils.ts
    UserProfile.test.tsx
    UserProfile.storybook.tsx
    test.utils.ts
    index.ts

hooks/
  useAuth/
    useAuth.ts
    useAuth.context.tsx
    useAuth.types.ts
    useAuth.constants.ts
    useAuth.test.ts
    index.ts
  useTheme/
    useTheme.ts
    useTheme.context.tsx
    useTheme.types.ts
    useTheme.test.ts
    index.ts
  useFormValidation/
    useFormValidation.ts
    useFormValidation.types.ts
    useFormValidation.test.ts
    index.ts
```

### Subdomain example

When a component domain has specialized variants:

```
DataTable/
  DataTable.tsx
  DataTable.types.ts
  index.ts
  SortableHeader/
    SortableHeader.tsx
    SortableHeader.types.ts
    SortableHeader.test.tsx
    index.ts
  Pagination/
    Pagination.tsx
    Pagination.types.ts
    Pagination.test.tsx
    index.ts
```

---

## Barrel Exports

Same rules as ts-patterns -- only `export` and `export type` statements, no logic. Both the container and render function are exported:

```ts
export { UserProfile, UserProfileRender } from "./UserProfile";

export type {
  UserProfileProps,
  UserProfileRenderProps,
} from "./UserProfile.types";
```

For context hooks, export the provider from the context file and the hook from the hook file:

```ts
export { AuthContextProvider } from "./useAuth.context";
export { useAuth } from "./useAuth";

export type { AuthContextType } from "./useAuth.types";
```

---

## Styling Conventions

### File convention

Each component gets a co-located `{Domain}.module.scss` file using CSS Modules for scoped class names. The component imports it as `styles`:

```tsx
import styles from "./UserProfile.module.scss";
```

### Snake_case class names

All CSS class names use **snake_case**. This keeps styling names visually distinct from TypeScript's camelCase/PascalCase and reads naturally in SCSS nesting:

```scss
.user_profile {
  display: flex;

  .avatar_container {
    flex-shrink: 0;
  }

  .status_indicator {
    flex: 1;
  }
}
```

When accessed in TypeScript via CSS Modules, use bracket notation to preserve the snake_case:

```tsx
<div className={styles["user_profile"]}>
  <div className={styles["avatar_container"]}>
```

### Tree-structured nesting

Component styles follow a tree structure that mirrors the DOM hierarchy. The root class matches the component name in snake_case, and child elements nest inside it. This keeps specificity low and makes the relationship between styles and markup easy to trace:

```scss
.user_profile {
  display: flex;
  gap: $spacing_md;

  .header {
    display: flex;
    align-items: center;
  }

  .content {
    display: flex;
    flex-direction: column;

    .bio {
      flex: 1;
    }

    .actions {
      display: flex;
      gap: $spacing_sm;
    }
  }
}
```

### Design system: global tokens and mixins

The project should maintain a global design system providing tokens (variables) and mixins that all components consume. The design system lives in a shared `styles/` directory at the project root:

```
styles/
  _tokens.scss
  _mixins.scss
  _breakpoints.scss
  _typography.scss
  global.scss
```

#### Tokens

Tokens define the primitive design values -- colors, spacing, typography scales, borders, shadows. Components import and use these rather than hardcoding values:

```scss
// styles/_tokens.scss

// Colors
$color_primary: #2563eb;
$color_primary_hover: #1d4ed8;
$color_surface: #ffffff;
$color_surface_alt: #f8fafc;
$color_text: #0f172a;
$color_text_muted: #64748b;
$color_border: #e2e8f0;
$color_error: #dc2626;
$color_success: #16a34a;

// Spacing
$spacing_xs: 4px;
$spacing_sm: 8px;
$spacing_md: 16px;
$spacing_lg: 24px;
$spacing_xl: 32px;
$spacing_2xl: 48px;

// Border radius
$radius_sm: 4px;
$radius_md: 8px;
$radius_lg: 12px;
$radius_full: 9999px;

// Shadows
$shadow_sm: 0 1px 2px rgba(0, 0, 0, 0.05);
$shadow_md: 0 4px 6px rgba(0, 0, 0, 0.1);
```

#### Mixins

Mixins encapsulate reusable patterns. Components `@include` them rather than repeating common property sets:

```scss
// styles/_mixins.scss

@mixin flex_center {
  display: flex;
  align-items: center;
  justify-content: center;
}

@mixin flex_column {
  display: flex;
  flex-direction: column;
}

@mixin truncate_text {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
```

#### Component usage

Components import the tokens and mixins they need:

```scss
@use "styles/tokens" as *;
@use "styles/mixins" as *;
@use "styles/breakpoints" as *;

.user_profile {
  @include flex_column;
  gap: $spacing_md;
  padding: $spacing_lg;
  background: $color_surface;
  border-radius: $radius_md;
}
```

**When to extend global tokens vs. keep local:** If a value is specific to one component (e.g., a particular icon size), define it locally in the component's `.module.scss`. If two or more components need the same value, promote it to the global tokens. The global design system should grow organically -- start lean and add tokens as shared patterns emerge.

### Flex-first layout

Use flexbox as the default layout mechanism. It handles alignment, distribution, and responsiveness cleanly without fragile positioning hacks.

```scss
// Good -- flex handles the layout
.nav_bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: $spacing_md;
}

// Avoid -- absolute positioning for something flex can handle
.nav_bar {
  position: relative;

  .logo {
    position: absolute;
    left: 16px;
  }

  .actions {
    position: absolute;
    right: 16px;
  }
}
```

**When `position: absolute` is appropriate:** Only use absolute (or fixed) positioning when elements genuinely need to overlap other content -- tooltips, dropdowns, modals, floating action buttons, notification badges. If you can achieve the layout with flex, use flex.

### Mobile-first responsive design

Styles are written for the smallest screen first, then progressively enhanced for larger viewports using `min-width` breakpoint mixins. This means the base styles (outside any breakpoint) target mobile, and you layer on complexity as the viewport grows.

#### Breakpoint tiers

The design system provides five standard breakpoints:

| Tier | Name | Min-width | Target devices |
|---|---|---|---|
| Base | (no mixin) | 0px | Small mobile (iPhone SE, etc.) |
| sm | `breakpoint_sm` | 375px | Regular mobile |
| md | `breakpoint_md` | 768px | Tablet |
| lg | `breakpoint_lg` | 1024px | Desktop |
| xl | `breakpoint_xl` | 1920px | Wide screen (4K, ultrawide) |

#### Breakpoint mixins

```scss
// styles/_breakpoints.scss

@mixin breakpoint_sm {
  @media (min-width: 375px) { @content; }
}

@mixin breakpoint_md {
  @media (min-width: 768px) { @content; }
}

@mixin breakpoint_lg {
  @media (min-width: 1024px) { @content; }
}

@mixin breakpoint_xl {
  @media (min-width: 1920px) { @content; }
}
```

#### Usage in components

Write the mobile layout first, then enhance:

```scss
@use "styles/tokens" as *;
@use "styles/breakpoints" as *;

.product_grid {
  display: flex;
  flex-direction: column;
  gap: $spacing_sm;

  @include breakpoint_sm {
    gap: $spacing_md;
  }

  @include breakpoint_md {
    flex-direction: row;
    flex-wrap: wrap;

    .product_card {
      flex: 0 0 calc(50% - $spacing_md);
    }
  }

  @include breakpoint_lg {
    .product_card {
      flex: 0 0 calc(33.333% - $spacing_md);
    }
  }

  @include breakpoint_xl {
    .product_card {
      flex: 0 0 calc(25% - $spacing_lg);
    }
  }
}
```

Never use `max-width` media queries for responsive design. The mobile-first approach means base styles are the small-screen styles, and breakpoints only add or override as the viewport expands.

---

## Testing

Tests use **Jest** as the test runner and **React Testing Library** (`@testing-library/react`) for rendering components and hooks. The core philosophy from ts-patterns still applies -- tests are behavioral specifications that define what *should* happen -- but React testing has its own patterns for interacting with the DOM and async state.

### Component testing

The container/render separation gives you two levels of testing. Test the render function to verify markup and visual behavior in isolation. Test the container to verify the full wiring with hooks and event handlers.

#### Testing the render function

The render function takes explicit props, so tests are straightforward -- no providers or mocked hooks needed:

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { UserProfileRender } from "./UserProfile";

describe("UserProfileRender", () => {
  it("should display the user's name and avatar", () => {
    render(
      <UserProfileRender
        displayName="Jane Doe"
        avatarUrl="https://example.com/avatar.jpg"
        isOnline={false}
        onLogout={jest.fn()}
      />,
    );

    expect(screen.getByText("Jane Doe")).toBeInTheDocument();
    expect(screen.getByAltText("Jane Doe")).toHaveAttribute(
      "src",
      "https://example.com/avatar.jpg",
    );
  });

  it("should show online indicator when user is online", () => {
    render(
      <UserProfileRender
        displayName="Jane Doe"
        avatarUrl="https://example.com/avatar.jpg"
        isOnline={true}
        onLogout={jest.fn()}
      />,
    );

    expect(screen.getByText("Online")).toBeInTheDocument();
  });

  it("should call onLogout when the logout button is clicked", async () => {
    const handleLogout = jest.fn();
    const user = userEvent.setup();

    render(
      <UserProfileRender
        displayName="Jane Doe"
        avatarUrl="https://example.com/avatar.jpg"
        isOnline={false}
        onLogout={handleLogout}
      />,
    );

    await user.click(screen.getByRole("button", { name: /log out/i }));

    expect(handleLogout).toHaveBeenCalledTimes(1);
  });
});
```

#### Testing the container

Container tests exercise the full component including its hooks. Mock external dependencies at the module level, and wrap with any required providers:

```tsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { UserProfile } from "./UserProfile";

jest.mock("../hooks/useAuth", () => ({
  useAuth: () => ({
    user: { displayName: "Jane Doe", avatarUrl: "https://example.com/avatar.jpg" },
    isOnline: true,
  }),
}));

const mockNavigate = jest.fn();
jest.mock("react-router-dom", () => ({
  ...jest.requireActual("react-router-dom"),
  useNavigate: () => mockNavigate,
}));

describe("UserProfile", () => {
  it("should navigate to login after logout", async () => {
    const user = userEvent.setup();

    render(<UserProfile userId="user-123" />);

    await user.click(screen.getByRole("button", { name: /log out/i }));

    expect(mockNavigate).toHaveBeenCalledWith("/login");
  });
});
```

### Hook testing

Test hooks using `renderHook` from `@testing-library/react`. This runs the hook inside a real React component lifecycle without you having to write a throwaway wrapper component. Use `act` to wrap state-changing operations so React processes updates before you assert:

```tsx
import { renderHook, act } from "@testing-library/react";
import { useAuth } from "./useAuth";

describe("useAuth", () => {
  it("should start in a loading state with no user", () => {
    const { result } = renderHook(() => useAuth());

    expect(result.current.isLoading).toBe(true);
    expect(result.current.currentUser).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });

  it("should update user after login", async () => {
    const { result } = renderHook(() => useAuth());

    await act(async () => {
      await result.current.login({
        email: "jane@example.com",
        password: "secure-password",
      });
    });

    expect(result.current.currentUser).toEqual(
      expect.objectContaining({ email: "jane@example.com" }),
    );
    expect(result.current.isAuthenticated).toBe(true);
  });

  it("should clear user after logout", async () => {
    const { result } = renderHook(() => useAuth());

    await act(async () => {
      await result.current.login({
        email: "jane@example.com",
        password: "secure-password",
      });
    });

    await act(async () => {
      await result.current.logout();
    });

    expect(result.current.currentUser).toBeNull();
    expect(result.current.isAuthenticated).toBe(false);
  });
});
```

#### Hooks with providers

When a hook depends on React context (e.g., a theme or router provider), pass a `wrapper` to `renderHook`:

```tsx
import { renderHook } from "@testing-library/react";
import { ThemeProvider } from "../providers/ThemeProvider";
import { useThemedStyles } from "./useThemedStyles";

function createWrapper({ children }: { children: React.ReactNode }) {
  return <ThemeProvider theme="dark">{children}</ThemeProvider>;
}

describe("useThemedStyles", () => {
  it("should return dark theme colors", () => {
    const { result } = renderHook(() => useThemedStyles(), {
      wrapper: createWrapper,
    });

    expect(result.current.backgroundColor).toBe("#1a1a2e");
  });
});
```

### Testing guidelines

1. **Query by role and accessible name first.** Use `screen.getByRole`, `screen.getByLabelText`, and `screen.getByPlaceholderText` before falling back to `screen.getByText` or `screen.getByTestId`. This ensures your tests verify accessibility as a side effect. Only use `data-testid` as a last resort when no semantic query applies.

2. **Use `userEvent` over `fireEvent`.** `userEvent.setup()` simulates real user interactions (typing, clicking, tabbing) more faithfully than `fireEvent`, which dispatches synthetic DOM events. This catches issues that `fireEvent` misses, like focus management and event ordering.

3. **Wrap state updates in `act`.** Any operation that triggers a React state update outside of a user interaction (like calling a hook's returned function directly, or simulating a timer) must be wrapped in `act` so React flushes updates before assertions run.

4. **Use `waitFor` for async outcomes.** When a component or hook triggers async work (data fetching, debounced updates), use `waitFor` to poll for the expected result rather than adding arbitrary delays:

    ```tsx
    import { waitFor } from "@testing-library/react";

    await waitFor(() => {
      expect(screen.getByText("Loaded")).toBeInTheDocument();
    });
    ```

5. **Test behavior, not implementation.** Don't assert on internal state, ref values, or how many times a hook re-rendered. Assert on what the user sees (DOM content, ARIA attributes) and what the component does (callbacks called, navigation triggered). If a test breaks because you refactored internals without changing behavior, the test was too coupled.

6. **Render function tests are your first line.** Because the render function takes explicit props, these tests are fast, stable, and cover the majority of visual behavior. Container tests are for verifying the glue -- that the right data reaches the render function and that user actions trigger the right side effects.

---

## What Still Applies from ts-patterns

Everything not overridden above carries over:

- **Functional architecture** -- factories over classes, composition over inheritance
- **Naming discipline** -- descriptive names, no abbreviations, minimum two words for locals
- **Type conventions** -- `readonly` for API contracts, discriminated unions, resolved companion types, no `any`
- **Comments and documentation** -- no file-level comments, JSDoc for public API, linter disables with reasons
- **Test philosophy** -- tests are behavioral specs, co-located as `{Domain}.test.tsx`, `describe`/`it` with "should" prefix. The Testing section above extends these principles with React Testing Library and Jest specifics
- **Detecting project context** -- adapt to existing project signals (linter, etc.)
