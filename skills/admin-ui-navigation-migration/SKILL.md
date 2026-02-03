---
name: admin-ui-navigation-migration
description: Migrate admin UI panels from internal navigation state to URL-based routing using AdminRouter. Use when asked to update navigation, implement routing, or migrate from redux navigation state to URL routing in admin panels.
---

# Admin UI Navigation to URL Routing Migration

## When to Use This Skill

- Migrating a panel from redux-based navigation to URL-based routing
- Implementing AdminRouter in a new or existing admin panel
- Fixing navigation issues related to Shell synchronization
- Adding URL parameter support to panel navigation

## Critical Migration Principle

**Preserve all existing navigation logic during migration.** Only the implementation changes—not the functionality. When migrating:

- Keep all existing routes and their purposes
- Preserve query parameters and their behavior
- Maintain the same navigation flows (list to detail to edit, etc.)
- Retain all conditional navigation logic

The old redux navigation code gets removed, but navigation behavior must remain identical.

## Core Principles

1. **AdminRouter is the single source of truth** - Never use react-router directly
2. **Routes must be centralized** - All route paths in `src/constants/routes.ts`
3. **Navigation stays synchronized with Shell** - Use AdminRouter APIs exclusively
4. **Base paths are dynamic** - Never hard-code panel base paths
5. **Permissions at route level** - No custom permission guards in components
6. **Prefer declarative over imperative** - Must use `AdminRouterLink` unless impossible
7. **Follow flat route naming** - Use `/<panel>/<action>/<id>` pattern

## Route Naming Patterns

**Flat navigation is preferred.** Use this pattern:

```
/<unique-panel-endpoint>/<action>
/<unique-panel-endpoint>/<action>/<id>
```

The `<unique-panel-endpoint>` must be WebAdminUrlMapping.YOUR_PANEL. Panels with common names (skills, users) may need more specific endpoints.

**Standard routes:**

```
/WebAdminUrlMapping.YOUR_PANEL                     # List view (root)
/WebAdminUrlMapping.YOUR_PANEL/create              # Create view
/WebAdminUrlMapping.YOUR_PANEL/edit/{id}           # Edit view
/WebAdminUrlMapping.YOUR_PANEL/edit/{id}?tab={tab} # Edit view with tab state
/WebAdminUrlMapping.YOUR_PANEL/view/{id}           # Detail/read-only view (if separate from edit)
```

**Nested sub-navigation example (internal panel navigation only, if tabs have internal routes we switch from query param above to sub-navigation below):**

```
/users/edit/{id}/skills
/users/edit/{id}/skills/assign
/users/edit/{id}/skills/assign/{skillId}
```

**Important:** If a proposed route structure doesn't follow this pattern (e.g., `/:id/edit` instead of `/edit/:id`), refactor it to comply before implementation.

## Architecture Overview

The Admin Shell is the source of truth for URL routing. Navigation flows through CompositeSDK:

1. Panel requests navigation via `AdminRouterLink` or `navigateTo()`
2. Request sent to Shell via CompositeSDK postMessage
3. Shell validates, updates browser URL, and sends route back to panel
4. Panel's internal MemoryRouter updates to match

This architecture ensures Shell and panel stay synchronized, enabling browser back/forward, bookmarking, and deep linking.

## Declarative vs Imperative Navigation

**Always prefer declarative navigation (AdminRouterLink) over imperative (navigateTo).** Declarative routing:

- Complies with accessibility (a11y) standards
- Renders proper anchor elements for screen readers
- Supports keyboard navigation and right-click context menus
- Enables browser prefetching hints

**Use imperative navigation only when declarative is impossible:**

- Programmatic redirects after form submission
- Navigation triggered by non-interactive events
- Accessibility-required focus management scenarios
- Navigation inside sagas or effects

### Declarative Routing (Preferred)

Use `AdminRouterLink` with Nimbus `renderAs` prop to maintain styling while enabling routing:

```typescript
import { AdminRouterLink, useAdminNavigate } from '@fivn/admin-ui-base';
import { Button, ButtonVariantEnum } from '@fivn/ui-components';

const MyComponent = () => {
  const { createPath } = useAdminNavigate();

  return (
    <Button
      variant={ButtonVariantEnum.link}
      label="Go to Skill Details"
      renderAs={<AdminRouterLink to={createPath({ id: '123456', params: { tab: 'general' } })} />}
    />
  );
};
```

### Imperative Routing (When Declarative Is Impossible)

Use `useAdminNavigate` for programmatic navigation in event handlers or effects:

```typescript
import { useAdminNavigate } from '@fivn/admin-ui-base';

const MyComponent = () => {
  const { navigateTo, navigateBack, navigateRoot, createPath } = useAdminNavigate();

  const handleFormSubmit = async () => {
    await saveData();
    navigateTo(createPath({ id: '123456', params: { tab: 'users' } }));
  };

  const handleCreate = () => {
    navigateTo(createPath({ action: 'create' }));
  };

  return (
    <form onSubmit={handleFormSubmit}>
      {/* Form content */}
      <button type="submit">Save</button>
      <button type="button" onClick={() => navigateBack()}>
        Cancel
      </button>
      <button type="button" onClick={() => navigateRoot()}>
        Back to List
      </button>
    </form>
  );
};
```

**Always use `createPath()` when constructing paths.** This helper ensures the panel's base path is correctly prepended, maintaining synchronization with the Shell.

### Path Helper Functions

`createPath` generates paths using `action || id`, producing either `/panel/create` OR `/panel/{id}`. For routes following the `/<action>/<id>` pattern (like `/edit/{id}`), create helper functions:

```typescript
// src/constants/routes.ts
const BASE = WebAdminUrlMapping.YOUR_PANEL;

export const ROUTES = {
  ROOT: `/${BASE}`,
  CREATE_VIEW: `/${BASE}/create`,
  EDIT_VIEW: `/${BASE}/edit/:id`,
  VIEW: `/${BASE}/view/:id`,
} as const;

// Path builders for dynamic routes
export const buildEditPath = (id: string): string => `/${BASE}/edit/${id}`;
export const buildViewPath = (id: string): string => `/${BASE}/view/${id}`;
```

**Usage:**

```tsx
import { buildEditPath } from 'constants/routes';

// Declarative - preferred (mandatory for compliance)
<Button label="Edit" renderAs={<AdminRouterLink to={buildEditPath(row.id)} />} />;

// Imperative - when declarative is absolutely impossible
navigateTo(buildEditPath(id));
```

## Parameter Handling

**Use `useAdminBase().navigation` for URL state, not react-router hooks directly.**

```typescript
import { useAdminBase } from '@fivn/admin-ui-base';
import { useParams } from 'react-router';

const MyComponent: FC = () => {
  const { navigation } = useAdminBase();
  const { id } = useParams<{ id: string }>(); // Path params from route definition

  // Query params: always memoize URLSearchParams
  const searchParams = useMemo(() => new URLSearchParams(navigation?.search || ''), [navigation?.search]);

  const tab = searchParams.get('tab') || 'general';
  const pathname = navigation?.pathname || '';
};
```

**Rules:**

- Path params (`:id`): Use `useParams()` from react-router
- Query params (`?tab=x`): Use `navigation.search` from `useAdminBase()`
- Pathname analysis: Use `navigation.pathname` from `useAdminBase()`
- Always wrap `URLSearchParams` in `useMemo` with `navigation?.search` dependency

## Migration Workflow

### Phase 1: Setup & Dependencies

#### 1.1 Verify Package Versions

Minimum required versions in `package.json`:

```json
{
  "dependencies": {
    "@fivn/admin-ui-base": "^4.4.7",
    "@fivn/composite-sdk": "^3.7.0",
    "@fivn/date-utils": "^2.4.0",
    "@fivn/design-tokens": "^4.38.0",
    "@fivn/icons": "2.25.0",
    "@fivn/nimbus": "^3.0.2",
    "@fivn/ui-components": "^4.187.1",
    "react-router-dom": "^6.20.0"
  },
  "devDependencies": {
    "@fivn/admin-dev-shell": "^1.29.1",
    "@types/react-router-dom": "^5.3.3"
  }
}
```

#### 1.2 Update config.dev.js

Define admin routes for local development following the `/<panel>/<action>/<id>` pattern:

```javascript
window.DevShell = {
  name: 'your-panel-name',
  permissions: ['resource.view', 'resource.edit'],
  features: ['admin-console.your-feature.temp'],
  adminRoutes: ['/your-panel', '/your-panel/create', '/your-panel/edit/:id', '/your-panel/view/:id'],
};
```

#### 1.3 Initialize Admin Base

Call at app entry point before any AdminRouter hooks:

```typescript
// src/index.tsx
import { initializeAdminBase, WebAdminUrlMapping } from '@fivn/admin-ui-base';

initializeAdminBase('your-panel-id', {
  panel: WebAdminUrlMapping.YOUR_PANEL,
});
```

### Phase 2: Create Route Constants

Create `src/constants/routes.ts` following the `/<panel>/<action>/<id>` pattern:

```typescript
import { WebAdminUrlMapping } from '@fivn/composite-sdk';

const BASE = WebAdminUrlMapping.YOUR_PANEL;

export const ROUTES = {
  LOCAL_DEV: '/', // Required for local development
  ROOT: `/${BASE}`, // List view
  CREATE_VIEW: `/${BASE}/create`,
  EDIT_VIEW: `/${BASE}/edit/:id`,
  VIEW: `/${BASE}/view/:id`, // Detail/read-only view
} as const;

// Path builders for routes with dynamic segments
export const buildEditPath = (id: string): string => `/${BASE}/edit/${id}`;
export const buildViewPath = (id: string): string => `/${BASE}/view/${id}`;
```

### Phase 3: Implement Router Configuration

Replace navigation controller with `useAdminRouter`:

**Before (Redux-based):**

```tsx
const App = () => {
  const { currentView, currentViewProps } = useNavigation();

  switch (currentView) {
    case NavigationView.LIST:
      return <ListView />;
    case NavigationView.DETAIL:
      return <DetailView {...currentViewProps} />;
  }
};
```

**After (AdminRouter):**

```tsx
import { useAdminRouter } from '@fivn/admin-ui-base';
import { ROUTES } from 'constants/routes';

const App = () => {
  const router = useAdminRouter({
    routes: [
      { path: ROUTES.LOCAL_DEV, element: <ListView />, permissions: ['resource.view'] },
      { path: ROUTES.ROOT, element: <ListView />, permissions: ['resource.view'] },
      { path: ROUTES.CREATE_VIEW, element: <CreateView />, permissions: ['resource.edit'] },
      { path: ROUTES.VIEW, element: <DetailView />, permissions: ['resource.view'] },
      { path: ROUTES.EDIT_VIEW, element: <EditView />, permissions: ['resource.view', 'resource.edit'] },
    ],
    notFound: <NotFoundView />,
  });

  return <Stack fullHeight>{router}</Stack>;
};
```

**Route order matters:** Place more specific routes (like `/create`) before parameterized routes (like `/edit/:id`) to ensure correct matching.

Routes with `permissions` array automatically enforce access control. Missing permissions display a restricted view.

### Phase 4: Migrate Navigation Logic

**Critical: Preserve all existing navigation behavior.** While removing redux navigation code, ensure all navigation paths, query params, and user flows are maintained in the new implementation.

#### 4.1 Map Existing Navigation to Routes

Document current navigation flows before migrating:

| Old Navigation                        | New Route            | Example Path                  |
| ------------------------------------- | -------------------- | ----------------------------- |
| `NavigationView.LIST`                 | `ROUTES.ROOT`        | `/users`                      |
| `NavigationView.CREATE`               | `ROUTES.CREATE_VIEW` | `/users/create`               |
| `NavigationView.DETAIL` with `{ id }` | `ROUTES.VIEW`        | `/users/view/123`             |
| `NavigationView.EDIT` with `{ id }`   | `ROUTES.EDIT_VIEW`   | `/users/edit/123?tab=general` |

#### 4.2 Replace Redux Navigation Actions

**Before:**

```tsx
const { redirectView } = useNavigation();
redirectView(NavigationView.DETAIL, { id: '123', tab: 'users' });
```

**After:**

```tsx
const { navigateTo, createPath } = useAdminNavigate();
navigateTo(createPath({ id: '123', params: { tab: 'users' } }));
```

#### 4.3 Update Saga Navigation

For existing redux-saga panels, import functions directly from admin-ui-base:

```typescript
import { navigateTo, navigateRoot } from '@fivn/admin-ui-base';
import { buildEditPath } from 'constants/routes';

export function* handleSaveItem({ payload }: PayloadAction<SaveRequest>) {
  try {
    const result = yield call(api.save, payload);
    yield call(navigateTo, buildEditPath(result.id));
  } catch (error) {
    yield call(navigateRoot);
  }
}
```

For new panels, prefer React hooks with a navigation service:

```typescript
// services/navigation.ts
import { buildViewPath, buildEditPath } from 'constants/routes';

type NavigateToFunction = (to: string) => void;
let navigateFunction: NavigateToFunction | null = null;

export const setNavigateFunction = (fn: NavigateToFunction): void => {
  navigateFunction = fn;
};

export const navigateToView = (id: string): void => {
  navigateFunction?.(buildViewPath(id));
};

export const navigateToEdit = (id: string): void => {
  navigateFunction?.(buildEditPath(id));
};

// In root component
const { navigateTo } = useAdminNavigate();
useEffect(() => setNavigateFunction(navigateTo), [navigateTo]);
```

### Phase 5: Update Component Navigation

#### 5.1 Convert to AdminRouterLink (Preferred)

**Before:**

```tsx
<Button
  variant={ButtonVariantEnum.link}
  label={row.name}
  onClick={() => redirectView(NavigationView.DETAIL, { id: row.id })}
/>
```

**After:**

```tsx
<Button
  variant={ButtonVariantEnum.link}
  label={row.name}
  renderAs={<AdminRouterLink to={createPath({ id: row.id, params: { tab: 'general' } })} />}
/>
```

#### 5.2 Replace Props with URL Parameters

**Before:**

```tsx
const DetailView: FC<{ itemId: string; tab?: string }> = ({ itemId, tab }) => {};
```

**After:**

```tsx
const DetailView: FC = () => {
  const { id } = useParams<{ id: string }>();
  const { navigation } = useAdminBase();
  const searchParams = useMemo(() => new URLSearchParams(navigation?.search || ''), [navigation?.search]);
  const tab = searchParams.get('tab') || 'general';
};
```

### Phase 6: Remove Redux Navigation Code

Delete these files:

- `src/redux/slices/navigation.ts` and tests
- `src/redux/hooks/useNavigation.ts` and tests
- `src/types/redux/navigation.ts`
- `src/types/model/navigation.ts`

Update `rootReducer.ts` and store types to remove navigation state.

### Phase 7: Update Tests

#### 7.1 Global Test Mocks

Add to `src/setupTests.ts`:

```typescript
jest.mock('@fivn/admin-ui-base', () => {
  const React = require('react');
  const actual = jest.requireActual('@fivn/admin-ui-base');

  return {
    ...actual,
    useAdminBase: jest.fn(() => ({
      domainId: 'test-domain-id',
      features: [],
      uiPermissions: [],
      userId: 'test-user-id',
      navigation: { search: '', pathname: '/' },
    })),
    useAdminNavigate: jest.fn(() => ({
      navigateTo: jest.fn(),
      navigateBack: jest.fn(),
      navigateRoot: jest.fn(),
      createPath: jest.fn((opts: any) => {
        const params = opts?.params || {};
        const queryString = new URLSearchParams(params).toString();
        const base = opts?.id ? `/panel/${opts.id}` : '/panel';
        return queryString ? `${base}?${queryString}` : base;
      }),
    })),
    useAdminRouter: jest.fn().mockImplementation((config: any) => {
      const mockFn = jest.requireMock('@fivn/admin-ui-base').useAdminBase;
      const navigation = mockFn.mock.results.slice(-1)[0]?.value?.navigation || {};
      const pathname = navigation.pathname || '/';
      const matched = config.routes?.find((r: any) => r.path === pathname);
      return matched?.element || config.routes?.[0]?.element || null;
    }),
    AdminRouterLink: React.forwardRef(({ to, children, ...props }: any, ref: any) =>
      React.createElement('a', { href: to, ref, ...props }, children),
    ),
  };
});
```

#### 7.2 Test Navigation Links

```typescript
it('renders navigation link with correct href', () => {
  render(<ListView />);
  const link = screen.getByRole('link', { name: 'View Details' });
  expect(link).toHaveAttribute('href', '/panel/123');
});
```

#### 7.3 Test with URL Params

```typescript
it('renders with URL params', () => {
  jest.spyOn(require('@fivn/admin-ui-base'), 'useAdminBase').mockReturnValue({
    navigation: { search: '?tab=users', pathname: '/panel/123' },
  });

  render(
    <MemoryRouter initialEntries={['/panel/123?tab=users']}>
      <Routes>
        <Route path="/panel/:id" element={<DetailView />} />
      </Routes>
    </MemoryRouter>,
  );

  expect(screen.getByText('Users Tab')).toBeInTheDocument();
});
```

## Cross-Panel Navigation

To navigate to a different panel, pass the full path directly:

```typescript
const { navigateTo } = useAdminNavigate();

// Navigate to a different panel
navigateTo('/skill-sets/123');
navigateTo('/users/agents');
```

The Shell detects the path belongs to a different panel, unmounts the current panel, and mounts the target panel.

## Common Patterns

### Tabs with URL State

```typescript
const DetailView = () => {
  const { id } = useParams<{ id: string }>();
  const { navigation } = useAdminBase();
  const { navigateTo, createPath } = useAdminNavigate();

  const searchParams = useMemo(() => new URLSearchParams(navigation?.search || ''), [navigation?.search]);
  const currentTab = searchParams.get('tab') || 'general';

  const handleTabChange = (tab: string) => {
    navigateTo(createPath({ id, params: { tab } }));
  };

  return (
    <Tabs value={currentTab} onChange={handleTabChange}>
      <Tab value="general">General</Tab>
      <Tab value="users">Users</Tab>
    </Tabs>
  );
};
```

### Back Navigation

```typescript
const DetailView = () => {
  const { navigateBack, navigateRoot } = useAdminNavigate();

  // navigateBack() pops local MemoryRouter history and syncs with Shell
  // navigateRoot() goes to panel root (/)

  return <Button onClick={() => navigateRoot()}>Back to List</Button>;
};
```

### Multi-Step Form

```typescript
const CreateForm = () => {
  const { navigation } = useAdminBase();
  const { navigateTo, createPath } = useAdminNavigate();

  const searchParams = useMemo(() => new URLSearchParams(navigation?.search || ''), [navigation?.search]);
  const step = parseInt(searchParams.get('step') || '1', 10);

  const goToStep = (newStep: number) => {
    navigateTo(createPath({ action: 'create', params: { step: String(newStep) } }));
  };
};
```

## Anti-Patterns

**Never use react-router directly:**

```typescript
// Wrong
import { Link, useNavigate } from 'react-router-dom';

// Correct
import { AdminRouterLink, useAdminNavigate } from '@fivn/admin-ui-base';
```

**Never hard-code paths without createPath:**

```typescript
// Wrong - breaks Shell sync
navigateTo(`/panel/${id}?tab=users`);

// Correct
navigateTo(createPath({ id, params: { tab: 'users' } }));
```

**Use path helpers for `/<action>/<id>` routes:**

```typescript
// createPath generates /panel/edit OR /panel/123, not both
// For /edit/123 pattern, use helper functions
buildEditPath('123'); // Returns /panel/edit/123
buildViewPath('123'); // Returns /panel/view/123
```

**Never implement custom permission guards:**

```typescript
// Wrong
if (!hasPermission('edit')) return <Restricted />;

// Correct - use route-level permissions
{ path: ROUTES.EDIT, element: <EditView />, permissions: ['edit'] }
```

**Never use onClick for navigation when a link is possible:**

```typescript
// Wrong - breaks a11y
<Button onClick={() => navigateTo(path)}>View</Button>

// Correct - accessible
<Button renderAs={<AdminRouterLink to={path} />}>View</Button>
```

## Validation Checklist

- [ ] Routes follow `/<panel>/<action>/<id>` naming pattern
- [ ] `initializeAdminBase()` called at app entry point
- [ ] All routes defined in `src/constants/routes.ts`
- [ ] `LOCAL_DEV: '/'` route registered
- [ ] `useAdminRouter` configured with all routes
- [ ] Declarative `AdminRouterLink` used for all clickable navigation
- [ ] Imperative `navigateTo` only for form submissions/effects
- [ ] Path helpers used for dynamic `/<action>/<id>` routes
- [ ] Redux navigation code removed
- [ ] All existing navigation flows preserved
- [ ] Tests updated with router mocks
- [ ] `config.dev.js` has `adminRoutes` array with correct patterns
- [ ] Permission-based routing configured
- [ ] 404 handling with `notFound` prop

## Success Criteria

A successful migration will:

1. Synchronize Shell and panel navigation state
2. Support browser back/forward navigation
3. Allow URL bookmarking and sharing
4. Maintain permission enforcement
5. Work in both local dev and production
6. Pass all existing and new tests
7. Have no console errors or warnings
8. Provide 404 handling for invalid routes
9. Support deep linking to specific views
10. Enable URL-based state management

## Troubleshooting

**Navigation not updating URL:**

- Verify `useAdminNavigate` is used, not `useNavigate`
- Check routes are defined in `useAdminRouter`

**404 errors in production:**

- Ensure `adminRoutes` in `config.dev.js` matches route paths
- Verify panel is registered in `WebAdminUrlMapping`

**Components not receiving URL params:**

- Use `useAdminBase().navigation.search` for query params, not `useLocation()`
- Wrap `URLSearchParams` in `useMemo` with `navigation?.search` dependency

**Shell not syncing with panel:**

- Verify `createPath()` is used for all navigation
- Check `initializeAdminBase()` is called before router hooks
