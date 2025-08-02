# React Router Setup for SharePoint Framework

This guide covers integrating React Router using **HashRouter** for client-side navigation within SharePoint Framework web parts, providing persistent URLs that work seamlessly within SharePoint's architecture.

## Why HashRouter for SPFx?

### HashRouter is the optimal routing solution for SharePoint Web Parts Framework:

1. **SharePoint Architecture Compatibility**

   - SharePoint controls server-side routing, while HashRouter operates entirely client-side
   - Hash-based routes (`#/path`) don't trigger server requests, avoiding conflicts with SharePoint's routing system

2. **No Server Configuration Required**

   - HashRouter works without any server-side configuration
   - Developers have limited control over SharePoint server settings in most environments

3. **URL Persistence**

   - Provides bookmarkable, shareable URLs within web parts
   - Maintains browser history for back/forward navigation
   - Preserves state when refreshing or sharing links

4. **Web Part Isolation**

   - Each web part can have its own routing without affecting other components
   - Multiple web parts on the same page can use independent routing

5. **SharePoint Security Boundaries**
   - Respects SharePoint's security model by not attempting to modify server routes
   - Works within the constraints of the SharePoint page lifecycle

## Installation

```bash
npm install react-router-dom
npm install -D @types/react-router-dom
```

## Basic Router Setup

### 1. App Router Component

Create `src/components/AppRouter.tsx`:

```typescript
import React from "react";
import {
  HashRouter as Router,
  Routes,
  Route,
  Navigate,
} from "react-router-dom";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { AppLayout } from "./layout/AppLayout";
import { DashboardPage } from "../pages/DashboardPage";
import { ProjectsPage } from "../pages/ProjectsPage";
import { ProjectDetailPage } from "../pages/ProjectDetailPage";
import { TasksPage } from "../pages/TasksPage";
import { SettingsPage } from "../pages/SettingsPage";
import { NotFoundPage } from "../pages/NotFoundPage";

type AppRouterProps = {
  context: WebPartContext;
};

export const AppRouter: React.FC<AppRouterProps> = ({ context }) => {
  return (
    <Router>
      <AppLayout context={context}>
        <Routes>
          {/* Default route */}
          <Route path="/" element={<Navigate to="/dashboard" replace />} />

          {/* Main routes */}
          <Route
            path="/dashboard"
            element={<DashboardPage context={context} />}
          />
          <Route
            path="/projects"
            element={<ProjectsPage context={context} />}
          />
          <Route
            path="/projects/:projectId"
            element={<ProjectDetailPage context={context} />}
          />
          <Route path="/tasks" element={<TasksPage context={context} />} />
          <Route
            path="/settings"
            element={<SettingsPage context={context} />}
          />

          {/* Catch-all route */}
          <Route path="*" element={<NotFoundPage />} />
        </Routes>
      </AppLayout>
    </Router>
  );
};
```

### 2. SharePoint Framework Integration

Update your web part file (e.g., `MyAppWebPart.ts`):

```typescript
import * as React from "react";
import * as ReactDom from "react-dom";
import { Version } from "@microsoft/sp-core-library";
import { BaseClientSideWebPart } from "@microsoft/sp-webpart-base";
import { AppRouter } from "./components/AppRouter";

export default class MyAppWebPart extends BaseClientSideWebPart<IMyAppWebPartProps> {
  public render(): void {
    const element: React.ReactElement = React.createElement(AppRouter, {
      context: this.context,
    });

    ReactDom.render(element, this.domElement);
  }

  protected onDispose(): void {
    ReactDom.unmountComponentAtNode(this.domElement);
  }

  protected get dataVersion(): Version {
    return Version.parse("1.0");
  }

  protected getPropertyPaneConfiguration(): IPropertyPaneConfiguration {
    return {
      pages: [
        {
          header: {
            description: strings.PropertyPaneDescription,
          },
          groups: [
            {
              groupName: strings.BasicGroupName,
              groupFields: [
                PropertyPaneTextField("description", {
                  label: strings.DescriptionFieldLabel,
                }),
              ],
            },
          ],
        },
      ],
    };
  }
}
```

## Navigation Components

### 1. Navigation Menu

Create `src/components/layout/Navigation.tsx`:

```typescript
import React from "react";
import { NavLink, useLocation } from "react-router-dom";
import { cn } from "@/lib/utils";
import { Home, FolderOpen, CheckSquare, Settings, Menu, X } from "lucide-react";
import { Button } from "@/components/ui/button";

type NavigationItem = {
  path: string;
  label: string;
  icon: React.ReactNode;
  description?: string;
};

const navigationItems: NavigationItem[] = [
  {
    path: "/dashboard",
    label: "Dashboard",
    icon: <Home className="h-4 w-4" />,
    description: "Overview and metrics",
  },
  {
    path: "/projects",
    label: "Projects",
    icon: <FolderOpen className="h-4 w-4" />,
    description: "Manage projects",
  },
  {
    path: "/tasks",
    label: "Tasks",
    icon: <CheckSquare className="h-4 w-4" />,
    description: "Track tasks",
  },
  {
    path: "/settings",
    label: "Settings",
    icon: <Settings className="h-4 w-4" />,
    description: "App configuration",
  },
];

type NavigationProps = {
  className?: string;
  onNavigate?: () => void;
};

export const Navigation: React.FC<NavigationProps> = ({
  className,
  onNavigate,
}) => {
  const location = useLocation();
  const [isMobileMenuOpen, setIsMobileMenuOpen] = React.useState(false);

  const toggleMobileMenu = () => {
    setIsMobileMenuOpen(!isMobileMenuOpen);
  };

  const handleNavClick = () => {
    setIsMobileMenuOpen(false);
    onNavigate?.();
  };

  return (
    <nav className={cn("bg-card border-r", className)}>
      {/* Mobile menu button */}
      <div className="md:hidden flex items-center justify-between p-4 border-b">
        <h2 className="font-semibold">Menu</h2>
        <Button variant="ghost" size="sm" onClick={toggleMobileMenu}>
          {isMobileMenuOpen ? (
            <X className="h-4 w-4" />
          ) : (
            <Menu className="h-4 w-4" />
          )}
        </Button>
      </div>

      {/* Navigation items */}
      <div
        className={cn(
          "space-y-1 p-4",
          isMobileMenuOpen ? "block" : "hidden md:block"
        )}
      >
        {navigationItems.map((item) => (
          <NavLink
            key={item.path}
            to={item.path}
            onClick={handleNavClick}
            className={({ isActive }) =>
              cn(
                "flex items-center gap-3 px-3 py-2 rounded-md text-sm font-medium transition-colors",
                "hover:bg-accent hover:text-accent-foreground",
                isActive
                  ? "bg-primary text-primary-foreground"
                  : "text-muted-foreground"
              )
            }
          >
            {item.icon}
            <div className="flex flex-col">
              <span>{item.label}</span>
              {item.description && (
                <span className="text-xs opacity-70">{item.description}</span>
              )}
            </div>
          </NavLink>
        ))}
      </div>
    </nav>
  );
};
```

### 2. Breadcrumb Navigation

Create `src/components/layout/Breadcrumb.tsx`:

```typescript
import React from "react";
import { useLocation, Link } from "react-router-dom";
import { ChevronRight, Home } from "lucide-react";
import { cn } from "@/lib/utils";

type BreadcrumbItem = {
  path: string;
  label: string;
  isActive?: boolean;
};

const getBreadcrumbItems = (pathname: string): BreadcrumbItem[] => {
  const segments = pathname.split("/").filter(Boolean);
  const items: BreadcrumbItem[] = [{ path: "/dashboard", label: "Home" }];

  let currentPath = "";
  segments.forEach((segment, index) => {
    currentPath += `/${segment}`;

    // Skip the first segment if it's dashboard (already added as Home)
    if (segment === "dashboard") return;

    let label = segment.charAt(0).toUpperCase() + segment.slice(1);

    // Handle dynamic routes
    if (segment.match(/^[a-f0-9\-]{36}$/i)) {
      // Looks like a GUID, treat as ID
      label = `Item ${segment.substring(0, 8)}...`;
    }

    items.push({
      path: currentPath,
      label,
      isActive: index === segments.length - 1,
    });
  });

  return items;
};

type BreadcrumbProps = {
  className?: string;
};

export const Breadcrumb: React.FC<BreadcrumbProps> = ({ className }) => {
  const location = useLocation();
  const breadcrumbItems = getBreadcrumbItems(location.pathname);

  return (
    <nav className={cn("flex items-center space-x-1 text-sm", className)}>
      {breadcrumbItems.map((item, index) => (
        <React.Fragment key={item.path}>
          {index > 0 && (
            <ChevronRight className="h-4 w-4 text-muted-foreground" />
          )}

          {item.isActive ? (
            <span className="font-medium text-foreground">{item.label}</span>
          ) : (
            <Link
              to={item.path}
              className="text-muted-foreground hover:text-foreground transition-colors"
            >
              {index === 0 ? <Home className="h-4 w-4" /> : item.label}
            </Link>
          )}
        </React.Fragment>
      ))}
    </nav>
  );
};
```

## Advanced Routing Patterns

### 1. Protected Routes

Create `src/components/routing/ProtectedRoute.tsx`:

```typescript
import React from "react";
import { Navigate, useLocation } from "react-router-dom";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { usePermissions } from "@/hooks/usePermissions";

type ProtectedRouteProps = {
  children: React.ReactNode;
  context: WebPartContext;
  requiredPermissions?: string[];
  requiredRoles?: string[];
  fallbackPath?: string;
};

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({
  children,
  context,
  requiredPermissions = [],
  requiredRoles = [],
  fallbackPath = "/dashboard",
}) => {
  const location = useLocation();
  const { hasPermissions, hasRoles, loading } = usePermissions({
    context,
    permissions: requiredPermissions,
    roles: requiredRoles,
  });

  if (loading) {
    return (
      <div className="flex items-center justify-center p-8">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
      </div>
    );
  }

  const hasAccess =
    (requiredPermissions.length === 0 || hasPermissions) &&
    (requiredRoles.length === 0 || hasRoles);

  if (!hasAccess) {
    return (
      <Navigate to={fallbackPath} state={{ from: location.pathname }} replace />
    );
  }

  return <>{children}</>;
};

// Usage in routes
export const AppRouterWithProtection: React.FC<{ context: WebPartContext }> = ({
  context,
}) => {
  return (
    <Router>
      <AppLayout context={context}>
        <Routes>
          <Route path="/" element={<Navigate to="/dashboard" replace />} />
          <Route
            path="/dashboard"
            element={<DashboardPage context={context} />}
          />
          <Route
            path="/projects"
            element={<ProjectsPage context={context} />}
          />

          {/* Protected route example */}
          <Route
            path="/settings"
            element={
              <ProtectedRoute
                context={context}
                requiredRoles={["Admin", "Owner"]}
              >
                <SettingsPage context={context} />
              </ProtectedRoute>
            }
          />

          <Route path="*" element={<NotFoundPage />} />
        </Routes>
      </AppLayout>
    </Router>
  );
};
```

### 2. Dynamic Routes with Parameters

Create `src/pages/ProjectDetailPage.tsx`:

```typescript
import React from "react";
import { useParams, useNavigate, Link } from "react-router-dom";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { ArrowLeft, Edit, Trash2 } from "lucide-react";
import { useProject } from "@/hooks/useProject";

type ProjectDetailPageProps = {
  context: WebPartContext;
};

export const ProjectDetailPage: React.FC<ProjectDetailPageProps> = ({
  context,
}) => {
  const { projectId } = useParams<{ projectId: string }>();
  const navigate = useNavigate();

  const { project, loading, error, deleteProject } = useProject({
    context,
    projectId: projectId || "",
  });

  const handleDelete = async () => {
    if (window.confirm("Are you sure you want to delete this project?")) {
      try {
        await deleteProject();
        navigate("/projects");
      } catch (error) {
        console.error("Failed to delete project:", error);
      }
    }
  };

  const handleEdit = () => {
    navigate(`/projects/${projectId}/edit`);
  };

  if (loading) {
    return (
      <div className="flex items-center justify-center p-8">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
      </div>
    );
  }

  if (error || !project) {
    return (
      <div className="p-4">
        <Card>
          <CardContent className="p-6">
            <p className="text-red-600">{error || "Project not found"}</p>
            <Link to="/projects">
              <Button variant="outline" className="mt-4">
                <ArrowLeft className="h-4 w-4 mr-2" />
                Back to Projects
              </Button>
            </Link>
          </CardContent>
        </Card>
      </div>
    );
  }

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <div className="flex items-center gap-4">
          <Link to="/projects">
            <Button variant="outline" size="sm">
              <ArrowLeft className="h-4 w-4 mr-2" />
              Back
            </Button>
          </Link>
          <div>
            <h1 className="text-2xl font-bold">{project.title}</h1>
            <p className="text-muted-foreground">Project Details</p>
          </div>
        </div>

        <div className="flex gap-2">
          <Button variant="outline" onClick={handleEdit}>
            <Edit className="h-4 w-4 mr-2" />
            Edit
          </Button>
          <Button variant="destructive" onClick={handleDelete}>
            <Trash2 className="h-4 w-4 mr-2" />
            Delete
          </Button>
        </div>
      </div>

      {/* Project Details */}
      <Card>
        <CardHeader>
          <CardTitle>Project Information</CardTitle>
        </CardHeader>
        <CardContent className="space-y-4">
          <div>
            <label className="text-sm font-medium text-muted-foreground">
              Description
            </label>
            <p className="mt-1">{project.description}</p>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="text-sm font-medium text-muted-foreground">
                Status
              </label>
              <p className="mt-1 capitalize">{project.status}</p>
            </div>

            <div>
              <label className="text-sm font-medium text-muted-foreground">
                Created
              </label>
              <p className="mt-1">
                {new Date(project.created).toLocaleDateString()}
              </p>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>
  );
};
```

## Custom Navigation Hooks

### 1. Navigation Helper Hook

Create `src/hooks/useNavigation.ts`:

```typescript
import { useNavigate, useLocation } from "react-router-dom";
import { useCallback } from "react";

type NavigationOptions = {
  replace?: boolean;
  state?: unknown;
};

export const useNavigation = () => {
  const navigate = useNavigate();
  const location = useLocation();

  const goTo = useCallback(
    (path: string, options?: NavigationOptions) => {
      navigate(path, {
        replace: options?.replace,
        state: options?.state,
      });
    },
    [navigate]
  );

  const goBack = useCallback(() => {
    navigate(-1);
  }, [navigate]);

  const goForward = useCallback(() => {
    navigate(1);
  }, [navigate]);

  const goToProject = useCallback(
    (projectId: string, options?: NavigationOptions) => {
      goTo(`/projects/${projectId}`, options);
    },
    [goTo]
  );

  const goToProjects = useCallback(
    (options?: NavigationOptions) => {
      goTo("/projects", options);
    },
    [goTo]
  );

  const goToDashboard = useCallback(
    (options?: NavigationOptions) => {
      goTo("/dashboard", options);
    },
    [goTo]
  );

  const isCurrentPath = useCallback(
    (path: string): boolean => {
      return location.pathname === path;
    },
    [location.pathname]
  );

  const getCurrentPath = useCallback(() => {
    return location.pathname;
  }, [location.pathname]);

  return {
    goTo,
    goBack,
    goForward,
    goToProject,
    goToProjects,
    goToDashboard,
    isCurrentPath,
    getCurrentPath,
    currentLocation: location,
  };
};
```

### 2. Route Parameters Hook

Create `src/hooks/useRouteParams.ts`:

```typescript
import { useParams, useSearchParams } from "react-router-dom";
import { useMemo } from "react";

export const useRouteParams = () => {
  const params = useParams();
  const [searchParams, setSearchParams] = useSearchParams();

  const queryParams = useMemo(() => {
    const result: Record<string, string> = {};
    searchParams.forEach((value, key) => {
      result[key] = value;
    });
    return result;
  }, [searchParams]);

  const setQueryParam = (key: string, value: string | null) => {
    if (value === null) {
      searchParams.delete(key);
    } else {
      searchParams.set(key, value);
    }
    setSearchParams(searchParams);
  };

  const removeQueryParam = (key: string) => {
    searchParams.delete(key);
    setSearchParams(searchParams);
  };

  const clearQueryParams = () => {
    setSearchParams({});
  };

  return {
    pathParams: params,
    queryParams,
    setQueryParam,
    removeQueryParam,
    clearQueryParams,
    searchParams,
    setSearchParams,
  };
};
```

## HashRouter URL Examples

With HashRouter, your URLs will look like:

```
https://yoursite.sharepoint.com/sites/yoursite/SitePages/YourPage.aspx#/dashboard
https://yoursite.sharepoint.com/sites/yoursite/SitePages/YourPage.aspx#/projects
https://yoursite.sharepoint.com/sites/yoursite/SitePages/YourPage.aspx#/projects/123e4567-e89b-12d3-a456-426614174000
https://yoursite.sharepoint.com/sites/yoursite/SitePages/YourPage.aspx#/tasks?filter=active&sort=dueDate
```

These URLs are:

- **Bookmarkable**: Users can save and share these URLs
- **Persistent**: Refreshing the page maintains the route
- **SharePoint-friendly**: Don't interfere with SharePoint's server routing
- **Isolated**: Each web part maintains its own routing context

## Best Practices

### 1. Route Organization

```typescript
// ✅ Good - Organize routes logically
const routes = {
  dashboard: "/dashboard",
  projects: {
    list: "/projects",
    detail: (id: string) => `/projects/${id}`,
    edit: (id: string) => `/projects/${id}/edit`,
    create: "/projects/new",
  },
  tasks: {
    list: "/tasks",
    detail: (id: string) => `/tasks/${id}`,
  },
} as const;
```

### 2. Route Guards

```typescript
// ✅ Good - Implement route guards for security
const useRouteGuard = (requiredRole: string) => {
  const { userRoles } = useCurrentUser();
  const navigate = useNavigation();

  useEffect(() => {
    if (!userRoles.includes(requiredRole)) {
      navigate.goToDashboard({ replace: true });
    }
  }, [userRoles, requiredRole, navigate]);
};
```

### 3. Lazy Loading

```typescript
// ✅ Good - Lazy load pages for better performance
const ProjectsPage = React.lazy(() => import("../pages/ProjectsPage"));
const TasksPage = React.lazy(() => import("../pages/TasksPage"));

export const AppRouter: React.FC<AppRouterProps> = ({ context }) => {
  return (
    <Router>
      <AppLayout context={context}>
        <Suspense fallback={<div>Loading...</div>}>
          <Routes>
            <Route
              path="/projects"
              element={<ProjectsPage context={context} />}
            />
            <Route path="/tasks" element={<TasksPage context={context} />} />
          </Routes>
        </Suspense>
      </AppLayout>
    </Router>
  );
};
```

## Next Steps

- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
- [Data Fetching Patterns](./08-data-fetching-patterns.md)
- [Component Architecture](./05-component-architecture.md)
