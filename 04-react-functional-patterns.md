# React Functional Component Patterns

This guide covers modern React functional component patterns for SharePoint Framework development, emphasizing hooks, functional programming, and const-based function declarations.

## Functional Component Structure

### Basic Component Pattern

Always use `const` declarations with arrow functions:

```typescript
import React from "react";

type MyComponentProps = {
  title: string;
  description?: string;
  className?: string;
};

export const MyComponent: React.FC<MyComponentProps> = ({
  title,
  description,
  className,
}) => {
  return (
    <div className={className}>
      <h2>{title}</h2>
      {description && <p>{description}</p>}
    </div>
  );
};
```

### Component with State and Effects

```typescript
import React, { useState, useEffect } from "react";

type DataItem = {
  id: string;
  name: string;
  value: number;
};

type DataListProps = {
  onDataLoad?: (data: DataItem[]) => void;
};

export const DataList: React.FC<DataListProps> = ({ onDataLoad }) => {
  const [data, setData] = useState<DataItem[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        setError(null);

        // Simulate API call
        const response = await fetch("/api/data");
        const result = await response.json();

        setData(result);
        onDataLoad?.(result);
      } catch (err) {
        setError(err instanceof Error ? err.message : "Unknown error");
      } finally {
        setLoading(false);
      }
    };

    loadData();
  }, [onDataLoad]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <ul>
      {data.map((item) => (
        <li key={item.id}>
          {item.name}: {item.value}
        </li>
      ))}
    </ul>
  );
};
```

## Custom Hooks Patterns

### SharePoint List Hook

```typescript
import { useState, useEffect, useCallback } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";

type UseSharePointListOptions = {
  listName: string;
  context: WebPartContext;
  autoLoad?: boolean;
};

type UseSharePointListResult<T> = {
  items: T[];
  loading: boolean;
  error: string | null;
  refresh: () => Promise<void>;
  create: (item: Partial<T>) => Promise<T>;
  update: (id: number, item: Partial<T>) => Promise<T>;
  delete: (id: number) => Promise<void>;
};

export const useSharePointList = <T extends { Id: number }>(
  options: UseSharePointListOptions
): UseSharePointListResult<T> => {
  const [items, setItems] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const { listName, context, autoLoad = true } = options;

  const loadItems = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);

      const list = context.pageContext.web.lists.getByTitle(listName);
      const response = await list.items.getAll();

      setItems(response as T[]);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to load items");
    } finally {
      setLoading(false);
    }
  }, [listName, context]);

  const createItem = useCallback(
    async (item: Partial<T>): Promise<T> => {
      try {
        const list = context.pageContext.web.lists.getByTitle(listName);
        const response = await list.items.add(item);

        await loadItems(); // Refresh list
        return response.data as T;
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to create item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, context, loadItems]
  );

  const updateItem = useCallback(
    async (id: number, item: Partial<T>): Promise<T> => {
      try {
        const list = context.pageContext.web.lists.getByTitle(listName);
        const response = await list.items.getById(id).update(item);

        await loadItems(); // Refresh list
        return response.data as T;
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to update item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, context, loadItems]
  );

  const deleteItem = useCallback(
    async (id: number): Promise<void> => {
      try {
        const list = context.pageContext.web.lists.getByTitle(listName);
        await list.items.getById(id).delete();

        await loadItems(); // Refresh list
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to delete item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, context, loadItems]
  );

  useEffect(() => {
    if (autoLoad) {
      loadItems();
    }
  }, [autoLoad, loadItems]);

  return {
    items,
    loading,
    error,
    refresh: loadItems,
    create: createItem,
    update: updateItem,
    delete: deleteItem,
  };
};
```

### Form State Hook

```typescript
import { useState, useCallback } from "react";

type ValidationRule<T> = (value: T) => string | null;

type UseFormStateOptions<T> = {
  initialValues: T;
  validationRules?: Partial<Record<keyof T, ValidationRule<T[keyof T]>>>;
  onSubmit: (values: T) => Promise<void>;
};

type UseFormStateResult<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isSubmitting: boolean;
  isValid: boolean;
  setValue: <K extends keyof T>(key: K, value: T[K]) => void;
  setFieldTouched: (key: keyof T) => void;
  handleSubmit: () => Promise<void>;
  reset: () => void;
};

export const useFormState = <T extends Record<string, unknown>>(
  options: UseFormStateOptions<T>
): UseFormStateResult<T> => {
  const { initialValues, validationRules, onSubmit } = options;

  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const validateField = useCallback(
    (key: keyof T, value: T[keyof T]): string | null => {
      const rule = validationRules?.[key];
      return rule ? rule(value) : null;
    },
    [validationRules]
  );

  const setValue = useCallback(
    <K extends keyof T>(key: K, value: T[K]) => {
      setValues((prev) => ({ ...prev, [key]: value }));

      // Validate field
      const error = validateField(key, value);
      setErrors((prev) => ({ ...prev, [key]: error }));
    },
    [validateField]
  );

  const setFieldTouched = useCallback((key: keyof T) => {
    setTouched((prev) => ({ ...prev, [key]: true }));
  }, []);

  const validateAllFields = useCallback((): boolean => {
    const newErrors: Partial<Record<keyof T, string>> = {};
    let hasErrors = false;

    Object.keys(values).forEach((key) => {
      const typedKey = key as keyof T;
      const error = validateField(typedKey, values[typedKey]);
      if (error) {
        newErrors[typedKey] = error;
        hasErrors = true;
      }
    });

    setErrors(newErrors);
    return !hasErrors;
  }, [values, validateField]);

  const handleSubmit = useCallback(async () => {
    // Mark all fields as touched
    const allTouched = Object.keys(values).reduce(
      (acc, key) => ({
        ...acc,
        [key]: true,
      }),
      {} as Partial<Record<keyof T, boolean>>
    );
    setTouched(allTouched);

    if (!validateAllFields()) {
      return;
    }

    try {
      setIsSubmitting(true);
      await onSubmit(values);
    } catch (error) {
      // Handle submission error
      console.error("Form submission failed:", error);
    } finally {
      setIsSubmitting(false);
    }
  }, [values, validateAllFields, onSubmit]);

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
    setIsSubmitting(false);
  }, [initialValues]);

  const isValid = Object.values(errors).every((error) => !error);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    isValid,
    setValue,
    setFieldTouched,
    handleSubmit,
    reset,
  };
};
```

## Event Handling Patterns

### Optimized Event Handlers

```typescript
import React, { useCallback, useMemo } from "react";

type ListItem = {
  id: string;
  name: string;
  completed: boolean;
};

type TodoListProps = {
  items: ListItem[];
  onItemToggle: (id: string) => void;
  onItemDelete: (id: string) => void;
};

export const TodoList: React.FC<TodoListProps> = ({
  items,
  onItemToggle,
  onItemDelete,
}) => {
  // Memoize filtered items
  const activeItems = useMemo(
    () => items.filter((item) => !item.completed),
    [items]
  );

  const completedItems = useMemo(
    () => items.filter((item) => item.completed),
    [items]
  );

  // Memoize event handlers to prevent unnecessary re-renders
  const handleToggle = useCallback(
    (id: string) => {
      onItemToggle(id);
    },
    [onItemToggle]
  );

  const handleDelete = useCallback(
    (id: string) => {
      onItemDelete(id);
    },
    [onItemDelete]
  );

  return (
    <div>
      <section>
        <h3>Active Items ({activeItems.length})</h3>
        {activeItems.map((item) => (
          <TodoItem
            key={item.id}
            item={item}
            onToggle={handleToggle}
            onDelete={handleDelete}
          />
        ))}
      </section>

      <section>
        <h3>Completed Items ({completedItems.length})</h3>
        {completedItems.map((item) => (
          <TodoItem
            key={item.id}
            item={item}
            onToggle={handleToggle}
            onDelete={handleDelete}
          />
        ))}
      </section>
    </div>
  );
};

// Memoized child component
const TodoItem: React.FC<{
  item: ListItem;
  onToggle: (id: string) => void;
  onDelete: (id: string) => void;
}> = React.memo(({ item, onToggle, onDelete }) => {
  const handleToggleClick = useCallback(() => {
    onToggle(item.id);
  }, [item.id, onToggle]);

  const handleDeleteClick = useCallback(() => {
    onDelete(item.id);
  }, [item.id, onDelete]);

  return (
    <div className="flex items-center gap-2 p-2">
      <input
        type="checkbox"
        checked={item.completed}
        onChange={handleToggleClick}
      />
      <span className={item.completed ? "line-through" : ""}>{item.name}</span>
      <button onClick={handleDeleteClick}>Delete</button>
    </div>
  );
});
```

## Data Fetching Patterns

### Async Data Component

```typescript
import React, { useState, useEffect } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";

type AsyncComponentState<T> =
  | { status: "idle"; data: null; error: null }
  | { status: "loading"; data: null; error: null }
  | { status: "success"; data: T; error: null }
  | { status: "error"; data: null; error: string };

type AsyncDataComponentProps<T> = {
  fetchData: () => Promise<T>;
  children: (state: AsyncComponentState<T>) => React.ReactNode;
  dependencies?: unknown[];
};

export const AsyncDataComponent = <T>({
  fetchData,
  children,
  dependencies = [],
}: AsyncDataComponentProps<T>) => {
  const [state, setState] = useState<AsyncComponentState<T>>({
    status: "idle",
    data: null,
    error: null,
  });

  useEffect(() => {
    let cancelled = false;

    const loadData = async () => {
      setState({ status: "loading", data: null, error: null });

      try {
        const data = await fetchData();

        if (!cancelled) {
          setState({ status: "success", data, error: null });
        }
      } catch (error) {
        if (!cancelled) {
          setState({
            status: "error",
            data: null,
            error: error instanceof Error ? error.message : "Unknown error",
          });
        }
      }
    };

    loadData();

    return () => {
      cancelled = true;
    };
  }, dependencies);

  return <>{children(state)}</>;
};

// Usage example
type Project = {
  id: string;
  title: string;
  description: string;
};

export const ProjectsPage: React.FC<{ context: WebPartContext }> = ({
  context,
}) => {
  const fetchProjects = async (): Promise<Project[]> => {
    const list = context.pageContext.web.lists.getByTitle("Projects");
    const items = await list.items.getAll();
    return items as Project[];
  };

  return (
    <div>
      <h1>Projects</h1>
      <AsyncDataComponent fetchData={fetchProjects}>
        {({ status, data, error }) => {
          switch (status) {
            case "loading":
              return <div>Loading projects...</div>;

            case "error":
              return <div>Error: {error}</div>;

            case "success":
              return (
                <div>
                  <p>Found {data?.length} projects</p>
                  {data?.map((project) => (
                    <div key={project.id}>
                      <h3>{project.title}</h3>
                      <p>{project.description}</p>
                    </div>
                  ))}
                </div>
              );

            default:
              return null;
          }
        }}
      </AsyncDataComponent>
    </div>
  );
};
```

## Error Boundary Pattern

```typescript
import React, { Component, ErrorInfo, ReactNode } from "react";

type ErrorBoundaryState = {
  hasError: boolean;
  error?: Error;
  errorInfo?: ErrorInfo;
};

type ErrorBoundaryProps = {
  children: ReactNode;
  fallback?: (error: Error, errorInfo: ErrorInfo) => ReactNode;
};

export class ErrorBoundary extends Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  constructor(props: ErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    this.setState({ error, errorInfo });

    // Log error to SharePoint or external service
    console.error("Error Boundary caught an error:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      if (this.props.fallback && this.state.error && this.state.errorInfo) {
        return this.props.fallback(this.state.error, this.state.errorInfo);
      }

      return (
        <div className="p-4 bg-red-50 border border-red-200 rounded">
          <h2 className="text-red-800 font-semibold">Something went wrong</h2>
          <details className="mt-2">
            <summary className="cursor-pointer text-red-600">
              Error details
            </summary>
            <pre className="mt-2 text-sm text-red-700 whitespace-pre-wrap">
              {this.state.error?.toString()}
            </pre>
          </details>
        </div>
      );
    }

    return this.props.children;
  }
}

// Functional Error Boundary Hook (React 18+)
export const useErrorBoundary = () => {
  const [error, setError] = useState<Error | null>(null);

  const resetError = useCallback(() => {
    setError(null);
  }, []);

  const captureError = useCallback((error: Error) => {
    setError(error);
  }, []);

  useEffect(() => {
    if (error) {
      throw error;
    }
  }, [error]);

  return { captureError, resetError };
};
```

## Performance Optimization Patterns

### Virtualized List Component

```typescript
import React, { useState, useMemo, useCallback } from "react";
import { FixedSizeList as List } from "react-window";

type VirtualizedListItem = {
  id: string;
  title: string;
  description: string;
};

type VirtualizedListProps = {
  items: VirtualizedListItem[];
  height: number;
  itemHeight: number;
  onItemClick?: (item: VirtualizedListItem) => void;
};

export const VirtualizedList: React.FC<VirtualizedListProps> = ({
  items,
  height,
  itemHeight,
  onItemClick,
}) => {
  const [searchTerm, setSearchTerm] = useState("");

  const filteredItems = useMemo(() => {
    if (!searchTerm) return items;

    return items.filter(
      (item) =>
        item.title.toLowerCase().includes(searchTerm.toLowerCase()) ||
        item.description.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [items, searchTerm]);

  const ItemRenderer = useCallback(
    ({ index, style }: { index: number; style: React.CSSProperties }) => {
      const item = filteredItems[index];

      return (
        <div
          style={style}
          className="flex items-center p-2 border-b hover:bg-gray-50 cursor-pointer"
          onClick={() => onItemClick?.(item)}
        >
          <div>
            <h4 className="font-medium">{item.title}</h4>
            <p className="text-sm text-gray-600">{item.description}</p>
          </div>
        </div>
      );
    },
    [filteredItems, onItemClick]
  );

  return (
    <div>
      <input
        type="text"
        placeholder="Search items..."
        value={searchTerm}
        onChange={(e) => setSearchTerm(e.target.value)}
        className="w-full p-2 mb-4 border rounded"
      />

      <List
        height={height}
        itemCount={filteredItems.length}
        itemSize={itemHeight}
        itemData={filteredItems}
      >
        {ItemRenderer}
      </List>
    </div>
  );
};
```

## Best Practices

### 1. Always Use Const for Function Components

```typescript
// ✅ Good
export const MyComponent: React.FC<Props> = ({ prop1, prop2 }) => {
  return <div>{prop1}</div>;
};

// ❌ Avoid
export function MyComponent({ prop1, prop2 }: Props) {
  return <div>{prop1}</div>;
}
```

### 2. Destructure Props in Function Signature

```typescript
// ✅ Good
export const UserCard: React.FC<{ user: User; onEdit: () => void }> = ({
  user,
  onEdit,
}) => {
  return (
    <div>
      <h3>{user.name}</h3>
      <button onClick={onEdit}>Edit</button>
    </div>
  );
};

// ❌ Less readable
export const UserCard: React.FC<{ user: User; onEdit: () => void }> = (
  props
) => {
  return (
    <div>
      <h3>{props.user.name}</h3>
      <button onClick={props.onEdit}>Edit</button>
    </div>
  );
};
```

### 3. Use Custom Hooks for Reusable Logic

```typescript
// ✅ Good - Extract reusable logic
const useToggle = (initialValue = false) => {
  const [value, setValue] = useState(initialValue);

  const toggle = useCallback(() => setValue((prev) => !prev), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);

  return { value, toggle, setTrue, setFalse };
};

export const CollapsibleSection: React.FC<{
  title: string;
  children: React.ReactNode;
}> = ({ title, children }) => {
  const { value: isOpen, toggle } = useToggle(false);

  return (
    <div>
      <button onClick={toggle}>{title}</button>
      {isOpen && <div>{children}</div>}
    </div>
  );
};
```

### 4. Memoize Expensive Calculations

```typescript
// ✅ Good - Memoize expensive operations
export const DataAnalysis: React.FC<{ data: number[] }> = ({ data }) => {
  const statistics = useMemo(() => {
    const sum = data.reduce((acc, val) => acc + val, 0);
    const average = sum / data.length;
    const max = Math.max(...data);
    const min = Math.min(...data);

    return { sum, average, max, min };
  }, [data]);

  return (
    <div>
      <p>Sum: {statistics.sum}</p>
      <p>Average: {statistics.average}</p>
      <p>Max: {statistics.max}</p>
      <p>Min: {statistics.min}</p>
    </div>
  );
};
```

## Next Steps

- [Component Architecture](./05-component-architecture.md)
- [Routing Setup](./06-routing-setup.md)
- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
