---
name: admin-ui-navigation-migration
description: Migrate admin UI panels from internal navigation state to URL-based routing using AdminRouter. Use when asked to update navigation, implement routing, or migrate from redux navigation state to URL routing in admin panels.
---

# Admin UI Navigation to URL Routing Migration

## When to Use This Skill

Use this skill when:
- Migrating a panel from redux-based navigation to URL-based routing
- Implementing AdminRouter in a new or existing admin panel
- Fixing navigation issues related to Shell synchronization
- Adding URL parameter support to panel navigation
- Converting imperative navigation (redux actions) to declarative routing

## Core Principles

1. **AdminRouter is the single source of truth** - Never use react-router directly
2. **Routes must be centralized** - All route paths in `src/constants/routes.ts`
3. **Navigation stays synchronized with Shell** - Use AdminRouter APIs exclusively
4. **Base paths are dynamic** - Never hard-code panel base paths
5. **Permissions at route level** - No custom permission guards in components

## Parameter Handling (Critical Pattern)

**Always use `adminBase.navigation` for URL state—not react-router hooks directly.**

```typescript
import { useAdminBase } from '@fivn/admin-ui-base';
import { useParams } from 'react-router';

const MyComponent: FC<{ userIdProp?: string; userNameProp?: string }> = (props) => {
  const { navigation } = useAdminBase();
  const { id } = useParams<{ id: string }>(); // Path params from route definition
  
  // Derive state from pathname segments if needed
  const pathname = navigation?.pathname || '';
  const pathSegments = pathname.split('/').filter(Boolean);
  
  // Query params: always use useMemo + URLSearchParams
  const searchParams = useMemo(
    () => new URLSearchParams(navigation?.search || ''),
    [navigation?.search]
  );
  
  // Support both props and URL params (props take precedence)
  const userId = props.userIdProp || searchParams.get('userId') || '';
  const userName = props.userNameProp || searchParams.get('userName') || '';
  const tab = searchParams.get('tab') || 'general';
};
```

**Key rules:**
- Path params (`:id`): Use `useParams()` from react-router
- Query params (`?tab=x`): Use `navigation.search` from `useAdminBase()`
- Pathname analysis: Use `navigation.pathname` from `useAdminBase()`
- Always wrap `URLSearchParams` in `useMemo` with `navigation?.search` dependency

## Migration Workflow

### Phase 1: Setup & Dependencies

#### 1.1 Verify Package Versions

Ensure minimum versions in `package.json`:

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

Define admin routes for local development:

```javascript
window.DevShell = {
  name: 'your-panel-name',
  permissions: ['resource.view', 'resource.edit'],
  features: ['admin-console.your-feature.temp'],
  adminRoutes: ['/your-panel', '/your-panel/:id', '/your-panel/:id/:subpage'],
};
```

#### 1.3 Initialize Admin Base

Before using AdminRouter, initialize Admin Base with your panel configuration. This ensures base paths and Shell synchronization work correctly.

```typescript
// src/index.tsx or app entry point
import { initializeAdminBase } from '@fivn/admin-ui-base';
import { WebAdminUrlMapping } from '@fivn/composite-sdk';

// Initialize with panel from WebAdminUrlMapping
initializeAdminBase('your-panel-id', {
  panel: WebAdminUrlMapping.YOUR_PANEL, // e.g., WebAdminUrlMapping.SKILLS
  // Other options as needed
});
```

**Important:** This initialization must happen before any AdminRouter hooks are called, typically at the application entry point.

### Phase 2: Create Route Constants

#### 2.1 Create src/constants/routes.ts

**Critical Structure Requirements:**
- Must include `LOCAL_DEV: '/'` for local development
- Panel routes must start with shell path (e.g., `/your-panel`)
- Base route should duplicate shell path for root view

```typescript
// src/constants/routes.ts
export const ROUTES = {
  LOCAL_DEV: '/', // Required for local development
  ROOT: '/your-panel', // Shell path + base route
  DETAIL_VIEW: '/your-panel/:id',
  EDIT_VIEW: '/your-panel/:id/edit',
} as const;

// Helper functions for path building (optional but recommended)
export const buildDetailPath = (id: string): string => {
  return `/your-panel/${id}`;
};

export const buildEditPath = (id: string): string => {
  return `/your-panel/${id}/edit`;
};
```

### Phase 3: Implement Router Configuration

#### 3.1 Update Root Component

Replace navigation controller with `useAdminRouter`:

**Before (Redux-based navigation):**
```tsx
// app/index.tsx
const App = () => {
  const { currentView, currentViewProps } = useNavigation();

  const renderView = () => {
    switch (currentView) {
      case NavigationView.LIST:
        return <ListView />;
      case NavigationView.DETAIL:
        return <DetailView {...currentViewProps} />;
    }
  };

  return <Stack fullHeight>{renderView()}</Stack>;
};
```

**After (AdminRouter):**
```tsx
// app/index.tsx or src/index.tsx
import { useAdminRouter } from '@fivn/admin-ui-base';
import { ROUTES } from 'constants/routes';

// Note: initializeAdminBase() should be called at app entry point
// before this component renders (see Phase 1.3)

const App = () => {
  const router = useAdminRouter({
    routes: [
      {
        path: ROUTES.LOCAL_DEV,
        element: <ListView />,
        permissions: ['resource.view'],
      },
      {
        path: ROUTES.ROOT,
        element: <ListView />,
        permissions: ['resource.view'],
      },
      {
        path: ROUTES.DETAIL_VIEW,
        element: <DetailView />,
        permissions: ['resource.view'],
      },
      {
        path: ROUTES.EDIT_VIEW,
        element: <EditView />,
        permissions: ['resource.view', 'resource.edit'],
      },
    ],
    notFound: <NotFoundView />,
  });

  return <Stack fullHeight>{router}</Stack>;
};
```

**Permission Handling:** Routes with `permissions` array automatically enforce access control. If a user lacks required permissions, AdminRouter displays a restricted view automatically—no custom guards needed in components.

### Phase 4: Update Navigation Calls

#### 4.1 Replace Redux Navigation Actions

**Before (Redux action dispatch):**
```tsx
const { redirectView } = useNavigation();

// Navigate to detail view
redirectView(NavigationView.DETAIL, { id: '123' });

// Navigate back
redirectView(NavigationView.LIST);
```

**After (AdminRouter hooks):**
```tsx
const { navigateTo, navigateRoot, createPath } = useAdminNavigate();

// Navigate to detail view - use helper function
navigateTo(buildDetailPath('123'));

// OR use createPath for dynamic navigation (recommended for Shell sync)
navigateTo(createPath({ id: '123' }));

// Navigate back to root
navigateRoot();
```

**Best Practice:** Always use `createPath()` for dynamic navigation with parameters. This ensures proper Shell synchronization and base path handling.

#### 4.2 Update Saga Navigation

**Before (Redux saga):**
```typescript
// sagas/items.ts
yield put(navigationActions.updateCurrentView({
  currentView: NavigationView.DETAIL,
  currentViewProps: { id: itemId }
}));
```

**After (Navigation service):**
```typescript
// services/navigation.ts
type NavigateToFunction = (to: string) => void;
let navigateFunction: NavigateToFunction | null = null;

export const setNavigateFunction = (navigate: NavigateToFunction): void => {
  navigateFunction = navigate;
};

export const navigateToDetail = (id: string): void => {
  if (navigateFunction) {
    navigateFunction(buildDetailPath(id));
  }
};

// Initialize in root component
const NavigationInitializer = ({ children }) => {
  const { navigateTo } = useAdminNavigate();
  
  useEffect(() => {
    setNavigateFunction(navigateTo);
  }, [navigateTo]);
  
  return <>{children}</>;
};

// In saga
import { navigateToDetail } from 'services/navigation';

yield call(navigateToDetail, itemId);
```

### Phase 5: Update Component Navigation

#### 5.1 Convert Links to AdminRouterLink

**Before (Button with onClick):**
```tsx
const handleClick = () => {
  redirectView(NavigationView.DETAIL, { id: row.id });
};

<Button
  variant={ButtonVariantEnum.link}
  label={row.name}
  onClick={handleClick}
/>
```

**After (AdminRouterLink with renderAs):**
```tsx
import { AdminRouterLink, useAdminNavigate } from '@fivn/admin-ui-base';

const { createPath } = useAdminNavigate();

<Button
  variant={ButtonVariantEnum.link}
  label={row.name}
  renderAs={
    <AdminRouterLink
      to={createPath({
        id: row.id,
        params: { tab: 'general' }
      })}
    />
  }
/>
```

**Why renderAs?** Using `renderAs` with Nimbus components preserves their styling and behavior while enabling routing functionality. `createPath()` ensures Shell synchronization and proper base path handling.

#### 5.2 Access URL Parameters

Replace props-based navigation state with URL parameters. See [Parameter Handling](#parameter-handling-critical-pattern) for the complete pattern.

**Before:**
```tsx
const DetailView: FC<{ itemId: string; tab?: string }> = ({ itemId, tab }) => {};
```

**After:**
```tsx
const DetailView: FC = () => {
  const { id } = useParams<{ id: string }>(); // Path param
  const { navigation } = useAdminBase();
  const searchParams = useMemo(
    () => new URLSearchParams(navigation?.search || ''),
    [navigation?.search]
  );
  const tab = searchParams.get('tab') || 'general'; // Query param
};
```

### Phase 6: Remove Redux Navigation Code

#### 6.1 Delete Navigation Slice

Remove files:
- `src/redux/slices/navigation.ts`
- `src/redux/slices/__tests__/navigation.test.ts`
- `src/redux/hooks/useNavigation.ts`
- `src/redux/hooks/__tests__/useNavigation.test.ts`
- `src/types/redux/navigation.ts`
- `src/types/model/navigation.ts` (if exists)

#### 6.2 Update Root Reducer

**Before:**
```typescript
// redux/rootReducer.ts
import navigationSlice from 'redux/slices/navigation';

const slices = [
  formsSlice,
  navigationSlice, // Remove this
  dataSlice
];
```

**After:**
```typescript
// redux/rootReducer.ts
const slices = [
  formsSlice,
  dataSlice
];
```

#### 6.3 Clean Up Store Types

**Before:**
```typescript
// types/redux/store.ts
export type RootState = {
  forms: FormsState;
  navigation: NavigationState; // Remove this
  data: DataState;
};
```

**After:**
```typescript
// types/redux/store.ts
export type RootState = {
  forms: FormsState;
  data: DataState;
};
```

### Phase 7: Update Tests

#### 7.1 Setup Test Mocks

Create global mocks in `src/setupTests.ts`:

```typescript
// setupTests.ts
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
      navigation: {
        search: '',
        pathname: '/',
      },
    })),
    useAdminNavigate: jest.fn(() => ({
      navigateTo: jest.fn(),
      navigateBack: jest.fn(),
      navigateRoot: jest.fn(),
      createPath: jest.fn((options: any) => {
        const base = options?.base || '/';
        const params = options?.params || {};
        const queryString = new URLSearchParams(params).toString();
        return queryString ? `${base}?${queryString}` : base;
      }),
    })),
    useAdminRouter: jest.fn().mockImplementation((config: any) => {
      const mockFn = jest.requireMock('@fivn/admin-ui-base').useAdminBase;
      const latestCall = mockFn.mock.results[mockFn.mock.results.length - 1];
      const navigation = latestCall?.value?.navigation || {};
      const pathname = navigation.pathname || '/';
      
      const matchedRoute = config.routes?.find((route: any) => route.path === pathname);
      return matchedRoute?.element || config.routes?.[0]?.element || null;
    }),
    AdminRouterLink: React.forwardRef(({ to, children, ...props }: any, ref: any) =>
      React.createElement('a', { href: to, ref, ...props }, children)
    ),
  };
});
```

#### 7.2 Update Component Tests

**Before (Testing with navigation mock):**
```typescript
import { mockUseNavigation } from 'utils/testing/mockUtils';

it('navigates to detail view', () => {
  const redirectViewMock = jest.fn();
  mockUseNavigation({ redirectView: redirectViewMock });
  
  render(<ListView />);
  fireEvent.click(screen.getByText('View Details'));
  
  expect(redirectViewMock).toHaveBeenCalledWith(
    NavigationView.DETAIL,
    { id: '123' }
  );
});
```

**After (Testing with router):**
```typescript
import { useAdminNavigate } from '@fivn/admin-ui-base';

it('navigates to detail view', () => {
  const navigateToMock = jest.fn();
  jest.spyOn(require('@fivn/admin-ui-base'), 'useAdminNavigate')
    .mockReturnValue({
      navigateTo: navigateToMock,
      navigateBack: jest.fn(),
      navigateRoot: jest.fn(),
      createPath: jest.fn((opts: any) => `/panel/${opts.id}`),
    });
  
  render(<ListView />);
  const link = screen.getByRole('link', { name: 'View Details' });
  
  expect(link).toHaveAttribute('href', '/panel/123');
});
```

#### 7.3 Test Components with URL Params

```typescript
import { MemoryRouter, Route, Routes } from 'react-router';

it('renders detail view with correct params', () => {
  const mockNavigateTo = jest.fn();
  
  jest.spyOn(require('@fivn/admin-ui-base'), 'useAdminBase')
    .mockReturnValue({
      navigation: {
        search: '?tab=users',
        pathname: '/panel/123',
      },
    });
  
  render(
    <MemoryRouter initialEntries={['/panel/123?tab=users']}>
      <Routes>
        <Route path="/panel/:id" element={<DetailView />} />
      </Routes>
    </MemoryRouter>
  );
  
  expect(screen.getByText('Users Tab')).toBeInTheDocument();
});
```

## Common Patterns & Solutions

### Pattern 1: Tabs with URL State

```typescript
const DetailView = () => {
  const { id } = useParams<{ id: string }>();
  const { navigation } = useAdminBase();
  const { navigateTo, createPath } = useAdminNavigate();
  
  const searchParams = useMemo(
    () => new URLSearchParams(navigation?.search || ''),
    [navigation?.search]
  );
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

### Pattern 2: Conditional Back Navigation

```typescript
const DetailView = () => {
  const { navigateBack, navigateRoot } = useAdminNavigate();
  const { navigation } = useAdminBase();
  
  const searchParams = useMemo(
    () => new URLSearchParams(navigation?.search || ''),
    [navigation?.search]
  );
  const fromPage = searchParams.get('from');
  
  const handleBack = () => fromPage ? navigateBack() : navigateRoot();
  
  return <Button onClick={handleBack}>Back</Button>;
};
```

### Pattern 3: Multi-Step Form

```typescript
const CreateForm = () => {
  const { navigation } = useAdminBase();
  const { navigateTo, createPath } = useAdminNavigate();
  
  const searchParams = useMemo(
    () => new URLSearchParams(navigation?.search || ''),
    [navigation?.search]
  );
  const step = parseInt(searchParams.get('step') || '1', 10);
  
  const goToStep = (newStep: number) => navigateTo(createPath({ params: { step: newStep } }));
  
  return (
    <Stepper currentStep={step}>
      <Step label="Details" />
      <Step label="Configuration" />
      <Step label="Review" />
    </Stepper>
  );
};
```

## Validation Checklist

Before completing migration, verify:

- [ ] `initializeAdminBase()` called at app entry point
- [ ] All routes defined in `src/constants/routes.ts`
- [ ] LOCAL_DEV route registered for local development
- [ ] `useAdminRouter` configured in root component
- [ ] All navigation uses `useAdminNavigate` or `AdminRouterLink`
- [ ] `createPath` used for **all** dynamic URLs (required for Shell sync)
- [ ] Redux navigation slice removed
- [ ] Navigation types removed from store
- [ ] Tests updated to use router mocks
- [ ] `config.dev.js` has `adminRoutes` array
- [ ] No direct usage of `react-router` APIs
- [ ] URL parameters replace navigation props
- [ ] Back/root navigation works correctly
- [ ] Permission-based routing works
- [ ] Not-found page renders for invalid routes

## Anti-Patterns to Avoid

❌ **Don't hard-code routes in components:**
```typescript
// Bad
navigateTo('/panel/123');

// Good
import { buildDetailPath } from 'constants/routes';
navigateTo(buildDetailPath('123'));
```

❌ **Don't use react-router directly:**
```typescript
// Bad
import { Link, useNavigate } from 'react-router-dom';

// Good
import { AdminRouterLink, useAdminNavigate } from '@fivn/admin-ui-base';
```

❌ **Don't pass navigation props:**
```typescript
// Bad
interface DetailViewProps {
  itemId: string;
}

// Good - use URL params
const DetailView = () => {
  const { id } = useParams<{ id: string }>();
};
```

❌ **Don't skip createPath for dynamic navigation:**
```typescript
// Bad - may break Shell sync
navigateTo(`/panel/${id}?tab=users`);

// Good - ensures Shell synchronization
navigateTo(createPath({ id, params: { tab: 'users' } }));
```

❌ **Don't implement custom permission guards:**
```typescript
// Bad
const DetailView = () => {
  if (!hasPermission('resource.edit')) {
    return <Restricted />;
  }
  return <Content />;
};

// Good - use route-level permissions
{
  path: ROUTES.DETAIL_VIEW,
  element: <DetailView />,
  permissions: ['resource.edit'],
}
```

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
- Check that routes are defined in `useAdminRouter`

**404 errors in production:**
- Ensure `adminRoutes` in `config.dev.js` matches route paths
- Verify base path configuration in Shell

**Tests failing:**
- Update `setupTests.ts` with AdminRouter mocks
- Use `MemoryRouter` for component tests with routing
- Mock `useAdminBase` to provide navigation state

**Components not receiving URL params:**
- Path params (`:id`): Use `useParams()` from react-router
- Query params: Use `useAdminBase().navigation.search`, NOT `useLocation()`
- Always wrap `URLSearchParams` in `useMemo` with `navigation?.search` dependency
- For pathname analysis, use `navigation.pathname.split('/').filter(Boolean)`