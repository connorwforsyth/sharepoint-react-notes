# Data Fetching Patterns

This guide covers advanced data fetching patterns, state management, and caching strategies for SharePoint Framework applications.

## React Query Integration

### Setup and Configuration

```bash
npm install @tanstack/react-query
npm install @tanstack/react-query-devtools
```

```typescript
// src/lib/queryClient.ts
import { QueryClient } from "@tanstack/react-query";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000, // 10 minutes (formerly cacheTime)
      retry: (failureCount, error) => {
        // Don't retry on 404s or permission errors
        if (error instanceof Error) {
          const httpError = error as any;
          if (httpError.status === 404 || httpError.status === 403) {
            return false;
          }
        }
        return failureCount < 3;
      },
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: 1,
    },
  },
});
```

### App Provider Setup

```typescript
// src/components/AppQueryProvider.tsx
import React from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { queryClient } from "@/lib/queryClient";

type AppQueryProviderProps = {
  children: React.ReactNode;
};

export const AppQueryProvider: React.FC<AppQueryProviderProps> = ({
  children,
}) => {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      {process.env.NODE_ENV === "development" && (
        <ReactQueryDevtools initialIsOpen={false} />
      )}
    </QueryClientProvider>
  );
};
```

## Query Hooks for SharePoint Data

### Projects Query Hooks

```typescript
// src/hooks/queries/useProjectQueries.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ProjectService } from "@/lib/sharePointService";
import {
  ProjectItem,
  ProjectItemCreate,
  ProjectItemUpdate,
} from "@/types/sharepoint";

const QUERY_KEYS = {
  projects: ["projects"] as const,
  project: (id: number) => ["projects", id] as const,
  projectsByStatus: (status: string) => ["projects", "status", status] as const,
  activeProjects: ["projects", "active"] as const,
};

// Get all projects
export const useProjects = (context: WebPartContext) => {
  const projectService = new ProjectService(context);

  return useQuery({
    queryKey: QUERY_KEYS.projects,
    queryFn: () => projectService.getAllProjects(),
    staleTime: 2 * 60 * 1000, // 2 minutes
  });
};

// Get single project
export const useProject = (context: WebPartContext, projectId: number) => {
  const projectService = new ProjectService(context);

  return useQuery({
    queryKey: QUERY_KEYS.project(projectId),
    queryFn: () => projectService.getListItemById("Projects", projectId),
    enabled: !!projectId,
  });
};

// Get projects by status
export const useProjectsByStatus = (
  context: WebPartContext,
  status: ProjectItem["Status"]
) => {
  const projectService = new ProjectService(context);

  return useQuery({
    queryKey: QUERY_KEYS.projectsByStatus(status),
    queryFn: () => projectService.getProjectsByStatus(status),
    enabled: !!status,
  });
};

// Create project mutation
export const useCreateProject = (context: WebPartContext) => {
  const queryClient = useQueryClient();
  const projectService = new ProjectService(context);

  return useMutation({
    mutationFn: (project: ProjectItemCreate) =>
      projectService.createProject(project),
    onSuccess: (newProject) => {
      // Invalidate and refetch projects list
      queryClient.invalidateQueries({ queryKey: QUERY_KEYS.projects });

      // Add to cache
      queryClient.setQueryData(QUERY_KEYS.project(newProject.Id), newProject);
    },
    onError: (error) => {
      console.error("Failed to create project:", error);
    },
  });
};

// Update project mutation
export const useUpdateProject = (context: WebPartContext) => {
  const queryClient = useQueryClient();
  const projectService = new ProjectService(context);

  return useMutation({
    mutationFn: ({ id, updates }: { id: number; updates: ProjectItemUpdate }) =>
      projectService.updateProject(id, updates),
    onSuccess: (updatedProject, { id }) => {
      // Update specific project in cache
      queryClient.setQueryData(QUERY_KEYS.project(id), updatedProject);

      // Update projects list cache
      queryClient.setQueryData(
        QUERY_KEYS.projects,
        (oldData: ProjectItem[] | undefined) => {
          if (!oldData) return oldData;
          return oldData.map((project) =>
            project.Id === id ? updatedProject : project
          );
        }
      );
    },
  });
};

// Delete project mutation
export const useDeleteProject = (context: WebPartContext) => {
  const queryClient = useQueryClient();
  const projectService = new ProjectService(context);

  return useMutation({
    mutationFn: (projectId: number) => projectService.deleteProject(projectId),
    onSuccess: (_, projectId) => {
      // Remove from cache
      queryClient.removeQueries({ queryKey: QUERY_KEYS.project(projectId) });

      // Update projects list cache
      queryClient.setQueryData(
        QUERY_KEYS.projects,
        (oldData: ProjectItem[] | undefined) => {
          if (!oldData) return oldData;
          return oldData.filter((project) => project.Id !== projectId);
        }
      );
    },
  });
};
```

### Optimistic Updates

```typescript
// src/hooks/queries/useOptimisticMutations.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ProjectService } from "@/lib/sharePointService";
import { ProjectItem, ProjectItemUpdate } from "@/types/sharepoint";

export const useOptimisticUpdateProject = (context: WebPartContext) => {
  const queryClient = useQueryClient();
  const projectService = new ProjectService(context);

  return useMutation({
    mutationFn: ({ id, updates }: { id: number; updates: ProjectItemUpdate }) =>
      projectService.updateProject(id, updates),

    // Optimistically update the UI
    onMutate: async ({ id, updates }) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: ["projects", id] });

      // Snapshot the previous value
      const previousProject = queryClient.getQueryData<ProjectItem>([
        "projects",
        id,
      ]);

      // Optimistically update to the new value
      if (previousProject) {
        queryClient.setQueryData<ProjectItem>(["projects", id], {
          ...previousProject,
          ...updates,
        });
      }

      return { previousProject };
    },

    // Rollback on error
    onError: (err, { id }, context) => {
      if (context?.previousProject) {
        queryClient.setQueryData(["projects", id], context.previousProject);
      }
    },

    // Always refetch after error or success
    onSettled: (data, error, { id }) => {
      queryClient.invalidateQueries({ queryKey: ["projects", id] });
    },
  });
};
```

## Advanced Data Fetching Patterns

### Infinite Queries

```typescript
// src/hooks/queries/useInfiniteProjects.ts
import { useInfiniteQuery } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ProjectService } from "@/lib/sharePointService";
import { ProjectItem } from "@/types/sharepoint";

type ProjectsPage = {
  projects: ProjectItem[];
  nextSkip: number | null;
  hasMore: boolean;
};

export const useInfiniteProjects = (
  context: WebPartContext,
  pageSize: number = 20
) => {
  const projectService = new ProjectService(context);

  return useInfiniteQuery({
    queryKey: ["projects", "infinite", pageSize],
    queryFn: async ({ pageParam = 0 }): Promise<ProjectsPage> => {
      const projects = await projectService.getListItems<ProjectItem>(
        "Projects",
        ["*", "ProjectManager/Title", "ProjectManager/Email"],
        ["ProjectManager"],
        undefined,
        "Created desc",
        pageSize,
        pageParam
      );

      return {
        projects,
        nextSkip: projects.length === pageSize ? pageParam + pageSize : null,
        hasMore: projects.length === pageSize,
      };
    },
    getNextPageParam: (lastPage) => lastPage.nextSkip,
    initialPageParam: 0,
  });
};

// Usage component
export const InfiniteProjectsList: React.FC<{ context: WebPartContext }> = ({
  context,
}) => {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    error,
  } = useInfiniteProjects(context);

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  const allProjects = data?.pages.flatMap((page) => page.projects) ?? [];

  return (
    <div>
      {allProjects.map((project) => (
        <div key={project.Id}>
          <h3>{project.Title}</h3>
          <p>{project.Description}</p>
        </div>
      ))}

      {hasNextPage && (
        <button onClick={() => fetchNextPage()} disabled={isFetchingNextPage}>
          {isFetchingNextPage ? "Loading more..." : "Load More"}
        </button>
      )}
    </div>
  );
};
```

### Parallel Queries

```typescript
// src/hooks/queries/useDashboardData.ts
import { useQueries } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ProjectService, TaskService } from "@/lib/sharePointService";

export const useDashboardData = (context: WebPartContext) => {
  const projectService = new ProjectService(context);
  const taskService = new TaskService(context);

  const results = useQueries({
    queries: [
      {
        queryKey: ["dashboard", "active-projects"],
        queryFn: () => projectService.getActiveProjects(),
        staleTime: 5 * 60 * 1000,
      },
      {
        queryKey: ["dashboard", "overdue-tasks"],
        queryFn: () => taskService.getOverdueTasks(),
        staleTime: 2 * 60 * 1000,
      },
      {
        queryKey: ["dashboard", "recent-projects"],
        queryFn: () =>
          projectService.getListItems(
            "Projects",
            ["*"],
            [],
            undefined,
            "Created desc",
            5
          ),
        staleTime: 10 * 60 * 1000,
      },
    ],
  });

  const [activeProjectsQuery, overdueTasksQuery, recentProjectsQuery] = results;

  return {
    activeProjects: {
      data: activeProjectsQuery.data,
      isLoading: activeProjectsQuery.isLoading,
      error: activeProjectsQuery.error,
    },
    overdueTasks: {
      data: overdueTasksQuery.data,
      isLoading: overdueTasksQuery.isLoading,
      error: overdueTasksQuery.error,
    },
    recentProjects: {
      data: recentProjectsQuery.data,
      isLoading: recentProjectsQuery.isLoading,
      error: recentProjectsQuery.error,
    },
    isLoading: results.some((query) => query.isLoading),
    hasError: results.some((query) => query.error),
  };
};
```

### Dependent Queries

```typescript
// src/hooks/queries/useProjectWithTasks.ts
import { useQuery } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ProjectService, TaskService } from "@/lib/sharePointService";

export const useProjectWithTasks = (
  context: WebPartContext,
  projectId: number
) => {
  const projectService = new ProjectService(context);
  const taskService = new TaskService(context);

  // Get project first
  const projectQuery = useQuery({
    queryKey: ["projects", projectId],
    queryFn: () => projectService.getListItemById("Projects", projectId),
    enabled: !!projectId,
  });

  // Get tasks only after project is loaded
  const tasksQuery = useQuery({
    queryKey: ["tasks", "project", projectId],
    queryFn: () => taskService.getTasksByProject(projectId),
    enabled: !!projectQuery.data?.Id,
  });

  return {
    project: projectQuery.data,
    tasks: tasksQuery.data,
    isLoading: projectQuery.isLoading || tasksQuery.isLoading,
    error: projectQuery.error || tasksQuery.error,
    projectLoading: projectQuery.isLoading,
    tasksLoading: tasksQuery.isLoading,
  };
};
```

## Custom Data Fetching Hooks

### Generic List Hook with React Query

```typescript
// src/hooks/queries/useSharePointQuery.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { SharePointService } from "@/lib/sharePointService";

type UseSharePointQueryOptions<T> = {
  listName: string;
  context: WebPartContext;
  select?: string[];
  expand?: string[];
  filter?: string;
  orderBy?: string;
  top?: number;
  enabled?: boolean;
  staleTime?: number;
};

export const useSharePointQuery = <T extends { Id: number }>(
  options: UseSharePointQueryOptions<T>
) => {
  const {
    listName,
    context,
    select,
    expand,
    filter,
    orderBy,
    top,
    enabled = true,
    staleTime = 5 * 60 * 1000,
  } = options;

  const service = new SharePointService(context);

  const queryKey = [
    "sharepoint",
    listName,
    { select, expand, filter, orderBy, top },
  ];

  return useQuery({
    queryKey,
    queryFn: () =>
      service.getListItems<T>(listName, select, expand, filter, orderBy, top),
    enabled,
    staleTime,
  });
};

// Generic mutation hooks
export const useSharePointMutation = <T extends { Id: number }>(
  listName: string,
  context: WebPartContext
) => {
  const queryClient = useQueryClient();
  const service = new SharePointService(context);

  const createMutation = useMutation({
    mutationFn: (item: Partial<T>) => service.createListItem<T>(listName, item),
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: ["sharepoint", listName],
      });
    },
  });

  const updateMutation = useMutation({
    mutationFn: ({ id, updates }: { id: number; updates: Partial<T> }) =>
      service.updateListItem<T>(listName, id, updates),
    onSuccess: (_, { id }) => {
      queryClient.invalidateQueries({
        queryKey: ["sharepoint", listName],
      });
      queryClient.invalidateQueries({
        queryKey: ["sharepoint", listName, id],
      });
    },
  });

  const deleteMutation = useMutation({
    mutationFn: (id: number) => service.deleteListItem(listName, id),
    onSuccess: (_, id) => {
      queryClient.invalidateQueries({
        queryKey: ["sharepoint", listName],
      });
      queryClient.removeQueries({
        queryKey: ["sharepoint", listName, id],
      });
    },
  });

  return {
    create: createMutation,
    update: updateMutation,
    delete: deleteMutation,
  };
};
```

## Background Sync and Offline Support

### Background Sync Hook

```typescript
// src/hooks/useBackgroundSync.ts
import { useEffect, useCallback } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { WebPartContext } from "@microsoft/sp-webpart-base";

export const useBackgroundSync = (context: WebPartContext) => {
  const queryClient = useQueryClient();

  const syncData = useCallback(async () => {
    try {
      // Refetch critical data in background
      await queryClient.refetchQueries({
        queryKey: ["projects"],
        type: "active",
      });

      await queryClient.refetchQueries({
        queryKey: ["tasks"],
        type: "active",
      });

      console.log("Background sync completed");
    } catch (error) {
      console.error("Background sync failed:", error);
    }
  }, [queryClient]);

  useEffect(() => {
    // Sync on window focus
    const handleFocus = () => {
      syncData();
    };

    // Sync every 10 minutes
    const interval = setInterval(syncData, 10 * 60 * 1000);

    window.addEventListener("focus", handleFocus);

    return () => {
      window.removeEventListener("focus", handleFocus);
      clearInterval(interval);
    };
  }, [syncData]);

  return { syncData };
};
```

### Offline Queue

```typescript
// src/lib/offlineQueue.ts
type QueuedMutation = {
  id: string;
  type: "create" | "update" | "delete";
  listName: string;
  data: any;
  timestamp: number;
};

class OfflineQueue {
  private queue: QueuedMutation[] = [];
  private storageKey = "spfx-offline-queue";

  constructor() {
    this.loadFromStorage();
  }

  private loadFromStorage(): void {
    try {
      const stored = localStorage.getItem(this.storageKey);
      if (stored) {
        this.queue = JSON.parse(stored);
      }
    } catch (error) {
      console.error("Failed to load offline queue:", error);
    }
  }

  private saveToStorage(): void {
    try {
      localStorage.setItem(this.storageKey, JSON.stringify(this.queue));
    } catch (error) {
      console.error("Failed to save offline queue:", error);
    }
  }

  add(mutation: Omit<QueuedMutation, "id" | "timestamp">): void {
    const queuedMutation: QueuedMutation = {
      ...mutation,
      id: Date.now().toString() + Math.random().toString(36).substr(2, 9),
      timestamp: Date.now(),
    };

    this.queue.push(queuedMutation);
    this.saveToStorage();
  }

  async processQueue(service: SharePointService): Promise<void> {
    const mutations = [...this.queue];
    this.queue = [];
    this.saveToStorage();

    for (const mutation of mutations) {
      try {
        switch (mutation.type) {
          case "create":
            await service.createListItem(mutation.listName, mutation.data);
            break;
          case "update":
            await service.updateListItem(
              mutation.listName,
              mutation.data.id,
              mutation.data.updates
            );
            break;
          case "delete":
            await service.deleteListItem(mutation.listName, mutation.data.id);
            break;
        }
      } catch (error) {
        console.error("Failed to process queued mutation:", error);
        // Re-add to queue for retry
        this.queue.push(mutation);
      }
    }

    this.saveToStorage();
  }

  getQueueSize(): number {
    return this.queue.length;
  }

  clear(): void {
    this.queue = [];
    this.saveToStorage();
  }
}

export const offlineQueue = new OfflineQueue();

// Hook to use offline queue
export const useOfflineQueue = (context: WebPartContext) => {
  const service = new SharePointService(context);

  const processQueue = useCallback(async () => {
    await offlineQueue.processQueue(service);
  }, [service]);

  const addToQueue = useCallback(
    (type: "create" | "update" | "delete", listName: string, data: any) => {
      offlineQueue.add({ type, listName, data });
    },
    []
  );

  return {
    processQueue,
    addToQueue,
    queueSize: offlineQueue.getQueueSize(),
    clearQueue: () => offlineQueue.clear(),
  };
};
```

## Error Handling and Retry Logic

### Query Error Boundary

```typescript
// src/components/QueryErrorBoundary.tsx
import React from "react";
import { QueryErrorResetBoundary } from "@tanstack/react-query";
import { ErrorBoundary } from "react-error-boundary";
import { Button } from "@/components/ui/Button";
import { AlertTriangle, RefreshCw } from "lucide-react";

type QueryErrorFallbackProps = {
  error: Error;
  resetErrorBoundary: () => void;
};

const QueryErrorFallback: React.FC<QueryErrorFallbackProps> = ({
  error,
  resetErrorBoundary,
}) => {
  return (
    <div className="flex flex-col items-center justify-center p-8 text-center">
      <AlertTriangle className="h-12 w-12 text-destructive mb-4" />
      <h2 className="text-xl font-semibold mb-2">Something went wrong</h2>
      <p className="text-muted-foreground mb-4 max-w-md">
        {error.message || "An unexpected error occurred while loading data."}
      </p>
      <Button onClick={resetErrorBoundary} className="gap-2">
        <RefreshCw className="h-4 w-4" />
        Try again
      </Button>
    </div>
  );
};

type QueryErrorBoundaryProps = {
  children: React.ReactNode;
};

export const QueryErrorBoundary: React.FC<QueryErrorBoundaryProps> = ({
  children,
}) => {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary
          FallbackComponent={QueryErrorFallback}
          onReset={reset}
          resetKeys={["query-error"]}
        >
          {children}
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
};
```

### Retry Logic with Exponential Backoff

```typescript
// src/lib/retryUtils.ts
export const exponentialBackoff = (
  attempt: number,
  baseDelay = 1000
): number => {
  return Math.min(baseDelay * Math.pow(2, attempt), 30000); // Max 30 seconds
};

export const createRetryOptions = () => ({
  retry: (failureCount: number, error: any) => {
    // Don't retry on certain HTTP errors
    const noRetryStatuses = [400, 401, 403, 404, 422];
    if (error?.status && noRetryStatuses.includes(error.status)) {
      return false;
    }

    // Retry up to 3 times
    return failureCount < 3;
  },
  retryDelay: (attemptIndex: number) => exponentialBackoff(attemptIndex),
});

// Usage in query configuration
export const queryClientWithRetry = new QueryClient({
  defaultOptions: {
    queries: {
      ...createRetryOptions(),
      staleTime: 5 * 60 * 1000,
      gcTime: 10 * 60 * 1000,
    },
    mutations: {
      retry: 1,
      retryDelay: 1000,
    },
  },
});
```

## Best Practices

### 1. Use Appropriate Cache Times

```typescript
// ✅ Good - Different cache times based on data volatility
const useProjects = () =>
  useQuery({
    queryKey: ["projects"],
    queryFn: fetchProjects,
    staleTime: 5 * 60 * 1000, // Projects change less frequently
  });

const useTasks = () =>
  useQuery({
    queryKey: ["tasks"],
    queryFn: fetchTasks,
    staleTime: 2 * 60 * 1000, // Tasks change more frequently
  });
```

### 2. Implement Loading States

```typescript
// ✅ Good - Comprehensive loading states
const ProjectsList: React.FC = () => {
  const { data, isLoading, isFetching, error } = useProjects();

  if (isLoading) return <ProjectsSkeleton />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <div>
      {isFetching && <LoadingIndicator />}
      <ProjectGrid projects={data} />
    </div>
  );
};
```

### 3. Optimize Mutations

```typescript
// ✅ Good - Optimistic updates with rollback
const useOptimisticToggle = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: updateTask,
    onMutate: async (variables) => {
      await queryClient.cancelQueries({ queryKey: ["tasks"] });
      const previous = queryClient.getQueryData(["tasks"]);

      queryClient.setQueryData(["tasks"], (old) =>
        optimisticallyUpdate(old, variables)
      );

      return { previous };
    },
    onError: (err, variables, context) => {
      queryClient.setQueryData(["tasks"], context?.previous);
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ["tasks"] });
    },
  });
};
```

### 4. Use Query Keys Consistently

```typescript
// ✅ Good - Structured query keys
const QUERY_KEYS = {
  all: ["data"] as const,
  projects: () => [...QUERY_KEYS.all, "projects"] as const,
  project: (id: number) => [...QUERY_KEYS.projects(), id] as const,
  projectTasks: (id: number) => [...QUERY_KEYS.project(id), "tasks"] as const,
} as const;
```

## Next Steps

- [SPFx API Usage](./09-spfx-api-usage.md)
- [Development Workflow](./10-development-workflow.md)
- [Performance Optimization](./12-performance-optimization.md)
