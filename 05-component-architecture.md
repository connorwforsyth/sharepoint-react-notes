# Component Architecture

This guide covers organizing and structuring components in your SharePoint Framework application for scalability, maintainability, and reusability.

## Directory Structure

### Recommended Project Structure

```
src/
├── components/
│   ├── ui/                     # shadcn/ui components
│   │   ├── Button.tsx
│   │   ├── Card.tsx
│   │   ├── Input.tsx
│   │   └── index.ts
│   ├── layout/                 # Layout components
│   │   ├── AppLayout.tsx
│   │   ├── Header.tsx
│   │   ├── Navigation.tsx
│   │   └── Sidebar.tsx
│   ├── forms/                  # Form components
│   │   ├── ProjectForm.tsx
│   │   ├── TaskForm.tsx
│   │   └── UserForm.tsx
│   ├── data/                   # Data display components
│   │   ├── ProjectList.tsx
│   │   ├── TaskTable.tsx
│   │   └── UserCard.tsx
│   └── common/                 # Shared components
│       ├── LoadingSpinner.tsx
│       ├── ErrorMessage.tsx
│       └── ConfirmDialog.tsx
├── hooks/                      # Custom hooks
│   ├── useSharePointList.ts
│   ├── useFormState.ts
│   └── useLocalStorage.ts
├── lib/                        # Utilities and helpers
│   ├── utils.ts
│   ├── api.ts
│   └── validation.ts
├── types/                      # Type definitions
│   ├── sharepoint.ts
│   ├── api.ts
│   └── common.ts
├── pages/                      # Page components
│   ├── ProjectsPage.tsx
│   ├── TasksPage.tsx
│   └── DashboardPage.tsx
└── styles/
    └── globals.css
```

## Component Categories

### 1. UI Components (Atomic Level)

Basic, reusable UI elements that compose into larger components:

```typescript
// src/components/ui/Button.tsx
import React from "react";
import { cn } from "@/lib/utils";
import { cva, type VariantProps } from "class-variance-authority";

const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
);

export type ButtonProps = React.ButtonHTMLAttributes<HTMLButtonElement> &
  VariantProps<typeof buttonVariants> & {
    asChild?: boolean;
  };

export const Button: React.FC<ButtonProps> = ({
  className,
  variant,
  size,
  asChild = false,
  ...props
}) => {
  return (
    <button
      className={cn(buttonVariants({ variant, size, className }))}
      {...props}
    />
  );
};
```

### 2. Layout Components

Components that define the structure and layout of your application:

```typescript
// src/components/layout/AppLayout.tsx
import React from "react";
import { Header } from "./Header";
import { Navigation } from "./Navigation";
import { Sidebar } from "./Sidebar";

type AppLayoutProps = {
  children: React.ReactNode;
  showSidebar?: boolean;
  currentPage?: string;
};

export const AppLayout: React.FC<AppLayoutProps> = ({
  children,
  showSidebar = true,
  currentPage,
}) => {
  return (
    <div className="min-h-screen bg-background">
      <Header />

      <div className="flex">
        <Navigation currentPage={currentPage} />

        <main className="flex-1 p-6">{children}</main>

        {showSidebar && <Sidebar />}
      </div>
    </div>
  );
};

// src/components/layout/Header.tsx
import React from "react";
import { Button } from "@/components/ui/Button";
import { User, Settings, Bell } from "lucide-react";

export const Header: React.FC = () => {
  return (
    <header className="border-b bg-background px-6 py-4">
      <div className="flex items-center justify-between">
        <div className="flex items-center space-x-4">
          <h1 className="text-xl font-semibold">My SharePoint App</h1>
        </div>

        <div className="flex items-center space-x-2">
          <Button variant="ghost" size="icon">
            <Bell className="h-4 w-4" />
          </Button>
          <Button variant="ghost" size="icon">
            <Settings className="h-4 w-4" />
          </Button>
          <Button variant="ghost" size="icon">
            <User className="h-4 w-4" />
          </Button>
        </div>
      </div>
    </header>
  );
};
```

### 3. Data Components

Components that handle data display and interaction:

```typescript
// src/components/data/ProjectList.tsx
import React from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/Card";
import { Button } from "@/components/ui/Button";
import { Badge } from "@/components/ui/Badge";
import { Edit, Trash2, Eye } from "lucide-react";

type Project = {
  Id: number;
  Title: string;
  Description: string;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  StartDate: Date;
  EndDate: Date;
  TeamSize: number;
};

type ProjectListProps = {
  projects: Project[];
  loading?: boolean;
  onView?: (project: Project) => void;
  onEdit?: (project: Project) => void;
  onDelete?: (project: Project) => void;
};

export const ProjectList: React.FC<ProjectListProps> = ({
  projects,
  loading = false,
  onView,
  onEdit,
  onDelete,
}) => {
  const getStatusVariant = (status: Project["Status"]) => {
    switch (status) {
      case "Completed":
        return "default";
      case "In Progress":
        return "secondary";
      case "Planning":
        return "outline";
      case "On Hold":
        return "destructive";
      default:
        return "secondary";
    }
  };

  if (loading) {
    return (
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {Array.from({ length: 6 }).map((_, i) => (
          <Card key={i} className="animate-pulse">
            <CardHeader>
              <div className="h-4 bg-gray-200 rounded w-3/4"></div>
              <div className="h-3 bg-gray-200 rounded w-1/2"></div>
            </CardHeader>
            <CardContent>
              <div className="space-y-2">
                <div className="h-3 bg-gray-200 rounded"></div>
                <div className="h-3 bg-gray-200 rounded w-5/6"></div>
              </div>
            </CardContent>
          </Card>
        ))}
      </div>
    );
  }

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
      {projects.map((project) => (
        <Card key={project.Id} className="hover:shadow-md transition-shadow">
          <CardHeader>
            <div className="flex items-start justify-between">
              <CardTitle className="text-lg">{project.Title}</CardTitle>
              <Badge variant={getStatusVariant(project.Status)}>
                {project.Status}
              </Badge>
            </div>
          </CardHeader>

          <CardContent>
            <p className="text-sm text-muted-foreground mb-4">
              {project.Description}
            </p>

            <div className="space-y-2 text-sm">
              <div className="flex justify-between">
                <span className="text-muted-foreground">Start Date:</span>
                <span>{project.StartDate.toLocaleDateString()}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-muted-foreground">End Date:</span>
                <span>{project.EndDate.toLocaleDateString()}</span>
              </div>
              <div className="flex justify-between">
                <span className="text-muted-foreground">Team Size:</span>
                <span>{project.TeamSize} members</span>
              </div>
            </div>

            <div className="flex justify-end space-x-2 mt-4 pt-4 border-t">
              {onView && (
                <Button
                  variant="outline"
                  size="sm"
                  onClick={() => onView(project)}
                >
                  <Eye className="h-4 w-4 mr-1" />
                  View
                </Button>
              )}
              {onEdit && (
                <Button
                  variant="outline"
                  size="sm"
                  onClick={() => onEdit(project)}
                >
                  <Edit className="h-4 w-4 mr-1" />
                  Edit
                </Button>
              )}
              {onDelete && (
                <Button
                  variant="destructive"
                  size="sm"
                  onClick={() => onDelete(project)}
                >
                  <Trash2 className="h-4 w-4 mr-1" />
                  Delete
                </Button>
              )}
            </div>
          </CardContent>
        </Card>
      ))}
    </div>
  );
};
```

### 4. Form Components

Components that handle form creation and validation:

```typescript
// src/components/forms/ProjectForm.tsx
import React from "react";
import { Button } from "@/components/ui/Button";
import { Input } from "@/components/ui/Input";
import { Textarea } from "@/components/ui/Textarea";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/Select";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/Card";
import { useFormState } from "@/hooks/useFormState";

type ProjectFormData = {
  Title: string;
  Description: string;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  StartDate: string;
  EndDate: string;
  TeamSize: number;
};

type ProjectFormProps = {
  initialData?: Partial<ProjectFormData>;
  onSubmit: (data: ProjectFormData) => Promise<void>;
  onCancel?: () => void;
  loading?: boolean;
};

export const ProjectForm: React.FC<ProjectFormProps> = ({
  initialData,
  onSubmit,
  onCancel,
  loading = false,
}) => {
  const form = useFormState({
    initialValues: {
      Title: initialData?.Title || "",
      Description: initialData?.Description || "",
      Status: initialData?.Status || "Planning",
      StartDate: initialData?.StartDate || "",
      EndDate: initialData?.EndDate || "",
      TeamSize: initialData?.TeamSize || 1,
    } as ProjectFormData,
    validationRules: {
      Title: (value) => {
        if (!value || value.length < 3) {
          return "Title must be at least 3 characters";
        }
        return null;
      },
      Description: (value) => {
        if (!value || value.length < 10) {
          return "Description must be at least 10 characters";
        }
        return null;
      },
      StartDate: (value) => {
        if (!value) {
          return "Start date is required";
        }
        return null;
      },
      EndDate: (value) => {
        if (!value) {
          return "End date is required";
        }
        if (
          form.values.StartDate &&
          new Date(value) <= new Date(form.values.StartDate)
        ) {
          return "End date must be after start date";
        }
        return null;
      },
      TeamSize: (value) => {
        if (value < 1) {
          return "Team size must be at least 1";
        }
        return null;
      },
    },
    onSubmit,
  });

  return (
    <Card className="w-full max-w-2xl mx-auto">
      <CardHeader>
        <CardTitle>
          {initialData ? "Edit Project" : "Create New Project"}
        </CardTitle>
      </CardHeader>

      <CardContent>
        <form
          onSubmit={(e) => {
            e.preventDefault();
            form.handleSubmit();
          }}
          className="space-y-4"
        >
          <div>
            <label htmlFor="title" className="block text-sm font-medium mb-1">
              Project Title
            </label>
            <Input
              id="title"
              value={form.values.Title}
              onChange={(e) => form.setValue("Title", e.target.value)}
              onBlur={() => form.setFieldTouched("Title")}
              placeholder="Enter project title"
              className={
                form.touched.Title && form.errors.Title ? "border-red-500" : ""
              }
            />
            {form.touched.Title && form.errors.Title && (
              <p className="text-red-500 text-sm mt-1">{form.errors.Title}</p>
            )}
          </div>

          <div>
            <label
              htmlFor="description"
              className="block text-sm font-medium mb-1"
            >
              Description
            </label>
            <Textarea
              id="description"
              value={form.values.Description}
              onChange={(e) => form.setValue("Description", e.target.value)}
              onBlur={() => form.setFieldTouched("Description")}
              placeholder="Enter project description"
              rows={3}
              className={
                form.touched.Description && form.errors.Description
                  ? "border-red-500"
                  : ""
              }
            />
            {form.touched.Description && form.errors.Description && (
              <p className="text-red-500 text-sm mt-1">
                {form.errors.Description}
              </p>
            )}
          </div>

          <div>
            <label htmlFor="status" className="block text-sm font-medium mb-1">
              Status
            </label>
            <Select
              value={form.values.Status}
              onValueChange={(value) =>
                form.setValue("Status", value as ProjectFormData["Status"])
              }
            >
              <SelectTrigger>
                <SelectValue placeholder="Select status" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="Planning">Planning</SelectItem>
                <SelectItem value="In Progress">In Progress</SelectItem>
                <SelectItem value="Completed">Completed</SelectItem>
                <SelectItem value="On Hold">On Hold</SelectItem>
              </SelectContent>
            </Select>
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label
                htmlFor="startDate"
                className="block text-sm font-medium mb-1"
              >
                Start Date
              </label>
              <Input
                id="startDate"
                type="date"
                value={form.values.StartDate}
                onChange={(e) => form.setValue("StartDate", e.target.value)}
                onBlur={() => form.setFieldTouched("StartDate")}
                className={
                  form.touched.StartDate && form.errors.StartDate
                    ? "border-red-500"
                    : ""
                }
              />
              {form.touched.StartDate && form.errors.StartDate && (
                <p className="text-red-500 text-sm mt-1">
                  {form.errors.StartDate}
                </p>
              )}
            </div>

            <div>
              <label
                htmlFor="endDate"
                className="block text-sm font-medium mb-1"
              >
                End Date
              </label>
              <Input
                id="endDate"
                type="date"
                value={form.values.EndDate}
                onChange={(e) => form.setValue("EndDate", e.target.value)}
                onBlur={() => form.setFieldTouched("EndDate")}
                className={
                  form.touched.EndDate && form.errors.EndDate
                    ? "border-red-500"
                    : ""
                }
              />
              {form.touched.EndDate && form.errors.EndDate && (
                <p className="text-red-500 text-sm mt-1">
                  {form.errors.EndDate}
                </p>
              )}
            </div>
          </div>

          <div>
            <label
              htmlFor="teamSize"
              className="block text-sm font-medium mb-1"
            >
              Team Size
            </label>
            <Input
              id="teamSize"
              type="number"
              min="1"
              value={form.values.TeamSize}
              onChange={(e) =>
                form.setValue("TeamSize", parseInt(e.target.value) || 1)
              }
              onBlur={() => form.setFieldTouched("TeamSize")}
              className={
                form.touched.TeamSize && form.errors.TeamSize
                  ? "border-red-500"
                  : ""
              }
            />
            {form.touched.TeamSize && form.errors.TeamSize && (
              <p className="text-red-500 text-sm mt-1">
                {form.errors.TeamSize}
              </p>
            )}
          </div>

          <div className="flex justify-end space-x-2 pt-4">
            {onCancel && (
              <Button
                type="button"
                variant="outline"
                onClick={onCancel}
                disabled={form.isSubmitting}
              >
                Cancel
              </Button>
            )}
            <Button
              type="submit"
              disabled={!form.isValid || form.isSubmitting || loading}
            >
              {form.isSubmitting || loading ? "Saving..." : "Save Project"}
            </Button>
          </div>
        </form>
      </CardContent>
    </Card>
  );
};
```

### 5. Page Components

High-level components that represent full pages or major sections:

```typescript
// src/pages/ProjectsPage.tsx
import React, { useState } from "react";
import { Button } from "@/components/ui/Button";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/Dialog";
import { ProjectList } from "@/components/data/ProjectList";
import { ProjectForm } from "@/components/forms/ProjectForm";
import { useSharePointList } from "@/hooks/useSharePointList";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { Plus } from "lucide-react";

type Project = {
  Id: number;
  Title: string;
  Description: string;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  StartDate: Date;
  EndDate: Date;
  TeamSize: number;
};

type ProjectsPageProps = {
  context: WebPartContext;
};

export const ProjectsPage: React.FC<ProjectsPageProps> = ({ context }) => {
  const [isCreateDialogOpen, setIsCreateDialogOpen] = useState(false);
  const [editingProject, setEditingProject] = useState<Project | null>(null);

  const {
    items: projects,
    loading,
    error,
    create,
    update,
    delete: deleteProject,
  } = useSharePointList<Project>({
    listName: "Projects",
    context,
  });

  const handleCreateProject = async (data: Omit<Project, "Id">) => {
    await create(data);
    setIsCreateDialogOpen(false);
  };

  const handleEditProject = async (data: Omit<Project, "Id">) => {
    if (editingProject) {
      await update(editingProject.Id, data);
      setEditingProject(null);
    }
  };

  const handleDeleteProject = async (project: Project) => {
    if (confirm(`Are you sure you want to delete "${project.Title}"?`)) {
      await deleteProject(project.Id);
    }
  };

  if (error) {
    return (
      <div className="p-6">
        <div className="bg-red-50 border border-red-200 rounded p-4">
          <h3 className="text-red-800 font-medium">Error Loading Projects</h3>
          <p className="text-red-600 mt-1">{error}</p>
        </div>
      </div>
    );
  }

  return (
    <div className="p-6">
      <div className="flex justify-between items-center mb-6">
        <div>
          <h1 className="text-2xl font-bold">Projects</h1>
          <p className="text-muted-foreground">
            Manage your organization's projects
          </p>
        </div>
        <Button onClick={() => setIsCreateDialogOpen(true)}>
          <Plus className="h-4 w-4 mr-2" />
          New Project
        </Button>
      </div>

      <ProjectList
        projects={projects}
        loading={loading}
        onEdit={setEditingProject}
        onDelete={handleDeleteProject}
      />

      {/* Create Project Dialog */}
      <Dialog open={isCreateDialogOpen} onOpenChange={setIsCreateDialogOpen}>
        <DialogContent className="max-w-2xl">
          <DialogHeader>
            <DialogTitle>Create New Project</DialogTitle>
          </DialogHeader>
          <ProjectForm
            onSubmit={handleCreateProject}
            onCancel={() => setIsCreateDialogOpen(false)}
          />
        </DialogContent>
      </Dialog>

      {/* Edit Project Dialog */}
      <Dialog
        open={!!editingProject}
        onOpenChange={() => setEditingProject(null)}
      >
        <DialogContent className="max-w-2xl">
          <DialogHeader>
            <DialogTitle>Edit Project</DialogTitle>
          </DialogHeader>
          {editingProject && (
            <ProjectForm
              initialData={editingProject}
              onSubmit={handleEditProject}
              onCancel={() => setEditingProject(null)}
            />
          )}
        </DialogContent>
      </Dialog>
    </div>
  );
};
```

## Component Composition Patterns

### Higher-Order Components (HOCs)

```typescript
// src/components/common/withLoading.tsx
import React from "react";

type WithLoadingProps = {
  loading?: boolean;
  loadingComponent?: React.ComponentType;
};

export const withLoading = <P extends object>(
  Component: React.ComponentType<P>
) => {
  const WithLoadingComponent: React.FC<P & WithLoadingProps> = ({
    loading = false,
    loadingComponent: LoadingComponent,
    ...props
  }) => {
    if (loading) {
      return LoadingComponent ? (
        <LoadingComponent />
      ) : (
        <div className="flex items-center justify-center p-8">
          <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
        </div>
      );
    }

    return <Component {...(props as P)} />;
  };

  WithLoadingComponent.displayName = `withLoading(${
    Component.displayName || Component.name
  })`;

  return WithLoadingComponent;
};

// Usage
const ProjectListWithLoading = withLoading(ProjectList);
```

### Render Props Pattern

```typescript
// src/components/common/DataProvider.tsx
import React from "react";

type DataProviderProps<T> = {
  fetchData: () => Promise<T>;
  children: (state: {
    data: T | null;
    loading: boolean;
    error: string | null;
    refetch: () => void;
  }) => React.ReactNode;
};

export const DataProvider = <T>({
  fetchData,
  children,
}: DataProviderProps<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const loadData = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      const result = await fetchData();
      setData(result);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Unknown error");
    } finally {
      setLoading(false);
    }
  }, [fetchData]);

  useEffect(() => {
    loadData();
  }, [loadData]);

  return <>{children({ data, loading, error, refetch: loadData })}</>;
};

// Usage
<DataProvider fetchData={fetchProjects}>
  {({ data, loading, error, refetch }) => (
    <ProjectList
      projects={data || []}
      loading={loading}
      error={error}
      onRefresh={refetch}
    />
  )}
</DataProvider>;
```

## Component Index Files

### Barrel Exports

Create index files to simplify imports:

```typescript
// src/components/ui/index.ts
export { Button } from "./Button";
export { Card, CardContent, CardHeader, CardTitle } from "./Card";
export { Input } from "./Input";
export { Badge } from "./Badge";
export { Dialog, DialogContent, DialogHeader, DialogTitle } from "./Dialog";

// src/components/forms/index.ts
export { ProjectForm } from "./ProjectForm";
export { TaskForm } from "./TaskForm";
export { UserForm } from "./UserForm";

// src/components/data/index.ts
export { ProjectList } from "./ProjectList";
export { TaskTable } from "./TaskTable";
export { UserCard } from "./UserCard";

// src/components/index.ts
export * from "./ui";
export * from "./forms";
export * from "./data";
export * from "./layout";
export * from "./common";
```

## Best Practices

### 1. Single Responsibility Principle

```typescript
// ✅ Good - Each component has a single responsibility
export const UserAvatar: React.FC<{
  user: User;
  size?: "sm" | "md" | "lg";
}> = ({ user, size = "md" }) => {
  // Only handles avatar display
};

export const UserProfile: React.FC<{ user: User }> = ({ user }) => {
  // Only handles profile information
};

export const UserActions: React.FC<{
  user: User;
  onEdit: () => void;
  onDelete: () => void;
}> = ({ user, onEdit, onDelete }) => {
  // Only handles user actions
};
```

### 2. Props Interface Design

```typescript
// ✅ Good - Clear, specific props
type DataTableProps<T> = {
  data: T[];
  columns: ColumnDefinition<T>[];
  loading?: boolean;
  pagination?: PaginationConfig;
  sorting?: SortingConfig<T>;
  filtering?: FilteringConfig<T>;
  onRowClick?: (item: T) => void;
  onSelectionChange?: (selectedItems: T[]) => void;
};
```

### 3. Component Composition over Inheritance

```typescript
// ✅ Good - Composition
export const ProjectCard: React.FC<ProjectCardProps> = ({ project }) => (
  <Card>
    <CardHeader>
      <ProjectTitle project={project} />
      <ProjectStatus status={project.status} />
    </CardHeader>
    <CardContent>
      <ProjectDescription description={project.description} />
      <ProjectMetadata project={project} />
    </CardContent>
    <CardFooter>
      <ProjectActions project={project} />
    </CardFooter>
  </Card>
);
```

### 4. Error Boundaries

```typescript
// Wrap page components with error boundaries
export const ProjectsPageWithErrorBoundary: React.FC<ProjectsPageProps> = (
  props
) => (
  <ErrorBoundary>
    <ProjectsPage {...props} />
  </ErrorBoundary>
);
```

## Next Steps

- [Routing Setup](./06-routing-setup.md)
- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
- [Data Fetching Patterns](./08-data-fetching-patterns.md)
