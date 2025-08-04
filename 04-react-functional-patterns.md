# React Functional Component Patterns for SharePoint Framework

This guide covers modern React functional component patterns for SharePoint Framework (SPFx) development, emphasizing hooks, functional programming, const-based function declarations, and integration with SharePoint lists using PnP JS library. All patterns follow TypeScript best practices and are optimized for Tailwind CSS and shadcn/ui component integration.

## Core Principles

### SharePoint Framework Context
- All components run in the context of the current SharePoint user
- Components are rendered in the normal page DOM (not isolated)
- Must be responsive and accessible by default
- Support full lifecycle management (render, load, serialize, deserialize)

## Functional Component Structure

### Basic Component Pattern

Always use `const` declarations with arrow functions and proper TypeScript typing:

```typescript
import React from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";

type MyComponentProps = {
  title: string;
  description?: string;
  context: WebPartContext;
  className?: string;
};

export const MyComponent: React.FC<MyComponentProps> = ({
  title,
  description,
  context,
  className,
}) => {
  return (
    <div className={`${className} p-4 bg-white rounded-lg shadow-sm`}>
      <h2 className="text-xl font-semibold text-gray-900">{title}</h2>
      {description && <p className="mt-2 text-gray-600">{description}</p>}
    </div>
  );
};
```

### WebPart Integration Pattern

```typescript
// MyWebPartWebPart.ts
import * as React from 'react';
import * as ReactDom from 'react-dom';
import { BaseClientSideWebPart } from '@microsoft/sp-webpart-base';
import { spfi, SPFx } from "@pnp/sp";
import "@pnp/sp/webs";
import "@pnp/sp/lists";
import "@pnp/sp/items";

import { MyComponent, IMyComponentProps } from './components/MyComponent';

export default class MyWebPartWebPart extends BaseClientSideWebPart<IMyWebPartWebPartProps> {
  protected async onInit(): Promise<void> {
    await super.onInit();
    
    // Initialize PnP JS with SPFx context
    const sp = spfi().using(SPFx(this.context));
  }

  public render(): void {
    const element: React.ReactElement<IMyComponentProps> = React.createElement(
      MyComponent,
      {
        title: this.properties.title,
        description: this.properties.description,
        context: this.context,
      }
    );

    ReactDom.render(element, this.domElement);
  }
}
```

### Component with State and Effects

```typescript
import React, { useState, useEffect } from "react";
import { SPFI } from "@pnp/sp";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { getSP } from "../services/pnpConfig";

type DataItem = {
  Id: number;
  Title: string;
  Description?: string;
  Status: string;
  Created: Date;
};

type DataListProps = {
  context: WebPartContext;
  listName: string;
  onDataLoad?: (data: DataItem[]) => void;
};

export const DataList: React.FC<DataListProps> = ({ 
  context, 
  listName,
  onDataLoad 
}) => {
  const [data, setData] = useState<DataItem[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        setError(null);

        const sp = getSP(context);
        const items = await sp.web.lists
          .getByTitle(listName)
          .items
          .select("Id", "Title", "Description", "Status", "Created")
          .orderBy("Created", false)
          .top(100)()
          .then(items => items as DataItem[]);

        setData(items);
        onDataLoad?.(items);
      } catch (err) {
        setError(err instanceof Error ? err.message : "Failed to load data");
        console.error("Error loading SharePoint list data:", err);
      } finally {
        setLoading(false);
      }
    };

    if (listName && context) {
      loadData();
    }
  }, [context, listName, onDataLoad]);

  if (loading) {
    return (
      <div className="flex items-center justify-center p-8">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600" />
      </div>
    );
  }

  if (error) {
    return (
      <div className="p-4 bg-red-50 border border-red-200 rounded-md">
        <p className="text-red-800">Error: {error}</p>
      </div>
    );
  }

  return (
    <div className="space-y-2">
      {data.map((item) => (
        <div key={item.Id} className="p-4 bg-white border rounded-lg shadow-sm hover:shadow-md transition-shadow">
          <h3 className="font-semibold text-gray-900">{item.Title}</h3>
          {item.Description && (
            <p className="mt-1 text-sm text-gray-600">{item.Description}</p>
          )}
          <div className="mt-2 flex items-center gap-4 text-xs text-gray-500">
            <span className="px-2 py-1 bg-blue-100 text-blue-800 rounded-full">
              {item.Status}
            </span>
            <span>{new Date(item.Created).toLocaleDateString()}</span>
          </div>
        </div>
      ))}
    </div>
  );
};
```

## PnP JS Configuration

### Service Configuration

```typescript
// services/pnpConfig.ts
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { spfi, SPFI, SPFx } from "@pnp/sp";
import "@pnp/sp/webs";
import "@pnp/sp/lists";
import "@pnp/sp/items";
import "@pnp/sp/batching";

let _sp: SPFI = null;

export const getSP = (context?: WebPartContext): SPFI => {
  if (_sp === null && context != null) {
    _sp = spfi().using(SPFx(context)).using({
      sp: {
        headers: {
          "Accept": "application/json; odata=verbose"
        }
      }
    });
  }
  return _sp;
};
```

## Custom Hooks Patterns

### SharePoint List Hook with Full CRUD Operations

```typescript
import { useState, useEffect, useCallback } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { getSP } from "../services/pnpConfig";
import { IItemAddResult, IItemUpdateResult } from "@pnp/sp/items";

type UseSharePointListOptions = {
  listName: string;
  context: WebPartContext;
  select?: string[];
  expand?: string[];
  filter?: string;
  orderBy?: string;
  orderByAscending?: boolean;
  top?: number;
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
  batchUpdate: (updates: { id: number; data: Partial<T> }[]) => Promise<void>;
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

      const sp = getSP(context);
      let query = sp.web.lists.getByTitle(listName).items;

      // Apply select fields
      if (select && select.length > 0) {
        query = query.select(...select);
      }

      // Apply expand fields for lookups
      if (expand && expand.length > 0) {
        query = query.expand(...expand);
      }

      // Apply filter
      if (filter) {
        query = query.filter(filter);
      }

      // Apply ordering
      if (orderBy) {
        query = query.orderBy(orderBy, orderByAscending ?? true);
      }

      // Apply top
      if (top) {
        query = query.top(top);
      }

      const response = await query();
      setItems(response as T[]);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to load items");
      console.error("Error in useSharePointList:", err);
    } finally {
      setLoading(false);
    }
  }, [listName, context, select, expand, filter, orderBy, orderByAscending, top]);

  const createItem = useCallback(
    async (item: Partial<T>): Promise<T> => {
      try {
        const sp = getSP(context);
        const response: IItemAddResult = await sp.web.lists
          .getByTitle(listName)
          .items
          .add(item);

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
        const sp = getSP(context);
        const response: IItemUpdateResult = await sp.web.lists
          .getByTitle(listName)
          .items
          .getById(id)
          .update(item);

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
        const sp = getSP(context);
        await sp.web.lists
          .getByTitle(listName)
          .items
          .getById(id)
          .delete();

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

  const batchUpdate = useCallback(
    async (updates: { id: number; data: Partial<T> }[]): Promise<void> => {
      try {
        const sp = getSP(context);
        const [batchedSP, execute] = sp.batched();
        
        const list = batchedSP.web.lists.getByTitle(listName);
        
        for (const update of updates) {
          list.items.getById(update.id).update(update.data);
        }
        
        await execute();
        await loadItems(); // Refresh list
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to batch update items";
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
    batchUpdate,
  };
};
```

### User Profile Hook

```typescript
import { useState, useEffect } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { getSP } from "../services/pnpConfig";
import "@pnp/sp/profiles";
import "@pnp/sp/site-users/web";

type UserProfile = {
  displayName: string;
  email: string;
  jobTitle?: string;
  department?: string;
  pictureUrl?: string;
  loginName: string;
};

export const useCurrentUser = (context: WebPartContext) => {
  const [user, setUser] = useState<UserProfile | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadUser = async () => {
      try {
        const sp = getSP(context);
        
        // Get current user
        const currentUser = await sp.web.currentUser();
        
        // Get user profile properties
        const profileProps = await sp.profiles.myProperties();
        
        const userProfile: UserProfile = {
          displayName: profileProps.DisplayName || currentUser.Title,
          email: profileProps.Email || currentUser.Email,
          jobTitle: profileProps.Title,
          department: profileProps.Department,
          pictureUrl: profileProps.PictureUrl,
          loginName: currentUser.LoginName
        };
        
        setUser(userProfile);
      } catch (err) {
        setError(err instanceof Error ? err.message : "Failed to load user profile");
      } finally {
        setLoading(false);
      }
    };

    if (context) {
      loadUser();
    }
  }, [context]);

  return { user, loading, error };
};
```

### Form State Hook with Validation

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

## SharePoint-Specific Components

### People Picker Component

```typescript
import React, { useState, useCallback } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { PeoplePicker, PrincipalType } from "@pnp/spfx-controls-react/lib/PeoplePicker";

type PeoplePickerFieldProps = {
  context: WebPartContext;
  label: string;
  selectedUsers?: string[];
  onChange: (users: string[]) => void;
  multiSelect?: boolean;
  required?: boolean;
  disabled?: boolean;
  placeholder?: string;
};

export const PeoplePickerField: React.FC<PeoplePickerFieldProps> = ({
  context,
  label,
  selectedUsers = [],
  onChange,
  multiSelect = false,
  required = false,
  disabled = false,
  placeholder = "Enter name or email..."
}) => {
  const [error, setError] = useState<string | null>(null);

  const handleSelectionChanged = useCallback((items: any[]) => {
    if (required && items.length === 0) {
      setError("This field is required");
    } else {
      setError(null);
    }
    
    const userEmails = items.map(item => item.secondaryText || item.loginName);
    onChange(userEmails);
  }, [onChange, required]);

  return (
    <div className="space-y-1">
      <label className="block text-sm font-medium text-gray-700">
        {label}
        {required && <span className="text-red-500 ml-1">*</span>}
      </label>
      <PeoplePicker
        context={context as any}
        personSelectionLimit={multiSelect ? 10 : 1}
        showtooltip={true}
        required={required}
        disabled={disabled}
        onChange={handleSelectionChanged}
        showHiddenInUI={false}
        principalTypes={[PrincipalType.User]}
        resolveDelay={1000}
        defaultSelectedUsers={selectedUsers}
        placeholder={placeholder}
      />
      {error && (
        <p className="mt-1 text-sm text-red-600">{error}</p>
      )}
    </div>
  );
};
```

### File Upload Component

```typescript
import React, { useState, useCallback } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { getSP } from "../services/pnpConfig";
import "@pnp/sp/webs";
import "@pnp/sp/files";
import "@pnp/sp/folders";

type FileUploadProps = {
  context: WebPartContext;
  libraryName: string;
  folderPath?: string;
  onUploadComplete?: (fileUrl: string) => void;
  acceptedFileTypes?: string;
  maxSizeMB?: number;
};

export const FileUpload: React.FC<FileUploadProps> = ({
  context,
  libraryName,
  folderPath = "",
  onUploadComplete,
  acceptedFileTypes = "*",
  maxSizeMB = 10
}) => {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  const [error, setError] = useState<string | null>(null);

  const handleFileUpload = useCallback(async (
    event: React.ChangeEvent<HTMLInputElement>
  ) => {
    const file = event.target.files?.[0];
    if (!file) return;

    // Validate file size
    const maxSizeBytes = maxSizeMB * 1024 * 1024;
    if (file.size > maxSizeBytes) {
      setError(`File size exceeds ${maxSizeMB}MB limit`);
      return;
    }

    try {
      setUploading(true);
      setError(null);
      setProgress(0);

      const sp = getSP(context);
      const fileName = file.name;
      const fileNamePath = encodeURI(fileName);
      
      let targetFolder;
      if (folderPath) {
        targetFolder = sp.web.getFolderByServerRelativePath(
          `${libraryName}/${folderPath}`
        );
      } else {
        targetFolder = sp.web.lists.getByTitle(libraryName).rootFolder;
      }

      // Upload file with progress tracking
      const fileContent = await file.arrayBuffer();
      
      const uploadResult = await targetFolder.files.addChunked(
        fileNamePath,
        fileContent,
        (data) => {
          const percentComplete = Math.round(
            (data.currentPointer / data.fileSize) * 100
          );
          setProgress(percentComplete);
        },
        true
      );

      const fileUrl = uploadResult.data.ServerRelativeUrl;
      onUploadComplete?.(fileUrl);
      
      // Reset input
      event.target.value = "";
    } catch (err) {
      setError(
        err instanceof Error ? err.message : "Failed to upload file"
      );
    } finally {
      setUploading(false);
      setProgress(0);
    }
  }, [context, libraryName, folderPath, maxSizeMB, onUploadComplete]);

  return (
    <div className="space-y-4">
      <div className="flex items-center justify-center w-full">
        <label className="flex flex-col items-center justify-center w-full h-32 border-2 border-gray-300 border-dashed rounded-lg cursor-pointer bg-gray-50 hover:bg-gray-100">
          <div className="flex flex-col items-center justify-center pt-5 pb-6">
            <svg className="w-8 h-8 mb-4 text-gray-500" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path strokeLinecap="round" strokeLinejoin="round" strokeWidth={2} d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
            </svg>
            <p className="mb-2 text-sm text-gray-500">
              <span className="font-semibold">Click to upload</span> or drag and drop
            </p>
            <p className="text-xs text-gray-500">
              Max file size: {maxSizeMB}MB
            </p>
          </div>
          <input
            type="file"
            className="hidden"
            onChange={handleFileUpload}
            accept={acceptedFileTypes}
            disabled={uploading}
          />
        </label>
      </div>
      
      {uploading && (
        <div className="w-full bg-gray-200 rounded-full h-2.5">
          <div 
            className="bg-blue-600 h-2.5 rounded-full transition-all duration-300"
            style={{ width: `${progress}%` }}
          />
        </div>
      )}
      
      {error && (
        <div className="p-3 bg-red-50 border border-red-200 rounded-md">
          <p className="text-sm text-red-800">{error}</p>
        </div>
      )}
    </div>
  );
};
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

## Integration with shadcn/ui Components

### SharePoint List with shadcn/ui Table

```typescript
import React from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Skeleton } from "@/components/ui/skeleton";
import { useSharePointList } from "../hooks/useSharePointList";

type TaskItem = {
  Id: number;
  Title: string;
  Status: "Not Started" | "In Progress" | "Completed";
  Priority: "Low" | "Normal" | "High";
  DueDate?: Date;
  AssignedToId?: number;
  AssignedTo?: {
    Title: string;
    EMail: string;
  };
};

type TasksTableProps = {
  context: WebPartContext;
  listName: string;
};

export const TasksTable: React.FC<TasksTableProps> = ({ context, listName }) => {
  const { items, loading, error, update } = useSharePointList<TaskItem>({
    context,
    listName,
    select: ["Id", "Title", "Status", "Priority", "DueDate", "AssignedTo/Title", "AssignedTo/EMail"],
    expand: ["AssignedTo"],
    orderBy: "DueDate",
    autoLoad: true
  });

  const handleStatusChange = async (taskId: number, newStatus: TaskItem["Status"]) => {
    try {
      await update(taskId, { Status: newStatus });
    } catch (error) {
      console.error("Failed to update status:", error);
    }
  };

  const getPriorityColor = (priority: TaskItem["Priority"]) => {
    switch (priority) {
      case "High": return "destructive";
      case "Normal": return "default";
      case "Low": return "secondary";
    }
  };

  const getStatusColor = (status: TaskItem["Status"]) => {
    switch (status) {
      case "Completed": return "success";
      case "In Progress": return "warning";
      case "Not Started": return "secondary";
    }
  };

  if (loading) {
    return (
      <div className="space-y-2">
        {[...Array(5)].map((_, i) => (
          <Skeleton key={i} className="h-12 w-full" />
        ))}
      </div>
    );
  }

  if (error) {
    return (
      <div className="text-center py-8">
        <p className="text-red-600">{error}</p>
      </div>
    );
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Title</TableHead>
          <TableHead>Status</TableHead>
          <TableHead>Priority</TableHead>
          <TableHead>Due Date</TableHead>
          <TableHead>Assigned To</TableHead>
          <TableHead>Actions</TableHead>
        </TableRow>
      </TableHeader>
      <TableBody>
        {items.map((task) => (
          <TableRow key={task.Id}>
            <TableCell className="font-medium">{task.Title}</TableCell>
            <TableCell>
              <Badge variant={getStatusColor(task.Status)}>
                {task.Status}
              </Badge>
            </TableCell>
            <TableCell>
              <Badge variant={getPriorityColor(task.Priority)}>
                {task.Priority}
              </Badge>
            </TableCell>
            <TableCell>
              {task.DueDate 
                ? new Date(task.DueDate).toLocaleDateString() 
                : "-"
              }
            </TableCell>
            <TableCell>
              {task.AssignedTo?.Title || "-"}
            </TableCell>
            <TableCell>
              <div className="flex gap-2">
                {task.Status !== "Completed" && (
                  <Button
                    size="sm"
                    variant="outline"
                    onClick={() => handleStatusChange(
                      task.Id, 
                      task.Status === "Not Started" ? "In Progress" : "Completed"
                    )}
                  >
                    {task.Status === "Not Started" ? "Start" : "Complete"}
                  </Button>
                )}
              </div>
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  );
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

## React Router v6 Integration for SPFx

### Router Setup with Hash Routing

```typescript
// App.tsx - Main app component with routing
import React from "react";
import { HashRouter, Routes, Route, Navigate } from "react-router-dom";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { SharePointProvider } from "./contexts/SharePointContext";
import { Layout } from "./components/Layout";
import { Dashboard } from "./pages/Dashboard";
import { ProjectList } from "./pages/ProjectList";
import { ProjectDetail } from "./pages/ProjectDetail";
import { Settings } from "./pages/Settings";

type AppProps = {
  context: WebPartContext;
};

// Use HashRouter for SPFx compatibility
export const App: React.FC<AppProps> = ({ context }) => {
  return (
    <SharePointProvider context={context}>
      <HashRouter>
        <Layout>
          <Routes>
            <Route path="/" element={<Navigate to="/dashboard" replace />} />
            <Route path="/dashboard" element={<Dashboard />} />
            <Route path="/projects" element={<ProjectList />} />
            <Route path="/projects/:id" element={<ProjectDetail />} />
            <Route path="/settings" element={<Settings />} />
          </Routes>
        </Layout>
      </HashRouter>
    </SharePointProvider>
  );
};
```

### Navigation Component with Active States

```typescript
import React from "react";
import { NavLink } from "react-router-dom";
import { cn } from "@/lib/utils";

const navigation = [
  { name: "Dashboard", href: "/dashboard", icon: "📊" },
  { name: "Projects", href: "/projects", icon: "📁" },
  { name: "Settings", href: "/settings", icon: "⚙️" },
];

export const Navigation: React.FC = () => {
  return (
    <nav className="flex space-x-4">
      {navigation.map((item) => (
        <NavLink
          key={item.name}
          to={item.href}
          className={({ isActive }) =>
            cn(
              "px-3 py-2 rounded-md text-sm font-medium transition-colors",
              isActive
                ? "bg-blue-100 text-blue-700"
                : "text-gray-700 hover:bg-gray-100"
            )
          }
        >
          <span className="mr-2">{item.icon}</span>
          {item.name}
        </NavLink>
      ))}
    </nav>
  );
};
```

### Protected Routes with SharePoint Permissions

```typescript
import React, { useEffect, useState } from "react";
import { Navigate, useLocation } from "react-router-dom";
import { useSharePoint } from "../contexts/SharePointContext";
import { PermissionKind } from "@pnp/sp/security";

type ProtectedRouteProps = {
  children: React.ReactNode;
  permission?: PermissionKind;
  redirectTo?: string;
};

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({ 
  children, 
  permission,
  redirectTo = "/dashboard" 
}) => {
  const { sp } = useSharePoint();
  const location = useLocation();
  const [hasPermission, setHasPermission] = useState<boolean | null>(null);

  useEffect(() => {
    const checkPermission = async () => {
      if (!permission) {
        setHasPermission(true);
        return;
      }

      try {
        const perms = await sp.web.currentUserHasPermissions(permission);
        setHasPermission(perms);
      } catch (error) {
        console.error("Permission check failed:", error);
        setHasPermission(false);
      }
    };

    checkPermission();
  }, [sp, permission]);

  if (hasPermission === null) {
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-blue-600" />
      </div>
    );
  }

  if (!hasPermission) {
    return <Navigate to={redirectTo} state={{ from: location }} replace />;
  }

  return <>{children}</>;
};
```

### Dynamic Route Parameters with SharePoint Data

```typescript
import React, { useEffect, useState } from "react";
import { useParams, useNavigate } from "react-router-dom";
import { useSharePoint } from "../contexts/SharePointContext";

type Project = {
  Id: number;
  Title: string;
  Description: string;
  Status: string;
  StartDate: Date;
  EndDate: Date;
};

export const ProjectDetail: React.FC = () => {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();
  const { sp } = useSharePoint();
  const [project, setProject] = useState<Project | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const loadProject = async () => {
      if (!id) return;

      try {
        const item = await sp.web.lists
          .getByTitle("Projects")
          .items
          .getById(parseInt(id))
          .select("Id", "Title", "Description", "Status", "StartDate", "EndDate")();
        
        setProject(item as Project);
      } catch (error) {
        console.error("Failed to load project:", error);
        navigate("/projects", { replace: true });
      } finally {
        setLoading(false);
      }
    };

    loadProject();
  }, [id, sp, navigate]);

  if (loading) {
    return <div className="animate-pulse">Loading project...</div>;
  }

  if (!project) {
    return <div>Project not found</div>;
  }

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">{project.Title}</h1>
        <button
          onClick={() => navigate("/projects")}
          className="px-4 py-2 text-sm bg-gray-200 rounded hover:bg-gray-300"
        >
          Back to Projects
        </button>
      </div>
      
      <div className="bg-white p-6 rounded-lg shadow">
        <p className="text-gray-600 mb-4">{project.Description}</p>
        
        <div className="grid grid-cols-2 gap-4">
          <div>
            <span className="font-semibold">Status:</span> {project.Status}
          </div>
          <div>
            <span className="font-semibold">Start Date:</span>{" "}
            {new Date(project.StartDate).toLocaleDateString()}
          </div>
        </div>
      </div>
    </div>
  );
};
```

### Navigation with Query Parameters

```typescript
import React, { useEffect } from "react";
import { useSearchParams, useNavigate } from "react-router-dom";
import { useSharePointList } from "../hooks/useSharePointList";

export const ProjectList: React.FC = () => {
  const [searchParams, setSearchParams] = useSearchParams();
  const navigate = useNavigate();
  
  const status = searchParams.get("status") || "all";
  const sortBy = searchParams.get("sortBy") || "Title";
  
  const filter = status === "all" ? undefined : `Status eq '${status}'`;
  
  const { items, loading } = useSharePointList({
    listName: "Projects",
    filter,
    orderBy: sortBy,
    autoLoad: true
  });

  const handleStatusFilter = (newStatus: string) => {
    setSearchParams(prev => {
      prev.set("status", newStatus);
      return prev;
    });
  };

  const handleSort = (field: string) => {
    setSearchParams(prev => {
      prev.set("sortBy", field);
      return prev;
    });
  };

  const handleProjectClick = (projectId: number) => {
    navigate(`/projects/${projectId}`);
  };

  return (
    <div>
      <div className="mb-4 flex gap-2">
        <select 
          value={status} 
          onChange={(e) => handleStatusFilter(e.target.value)}
          className="px-3 py-2 border rounded"
        >
          <option value="all">All Status</option>
          <option value="Active">Active</option>
          <option value="Completed">Completed</option>
          <option value="On Hold">On Hold</option>
        </select>
        
        <select 
          value={sortBy} 
          onChange={(e) => handleSort(e.target.value)}
          className="px-3 py-2 border rounded"
        >
          <option value="Title">Sort by Title</option>
          <option value="Created">Sort by Date</option>
          <option value="Status">Sort by Status</option>
        </select>
      </div>
      
      {/* Project list rendering */}
    </div>
  );
};
```

### Breadcrumb Navigation

```typescript
import React from "react";
import { Link, useLocation } from "react-router-dom";
import { ChevronRight } from "lucide-react";

export const Breadcrumbs: React.FC = () => {
  const location = useLocation();
  const pathnames = location.pathname.split("/").filter((x) => x);

  return (
    <nav className="flex mb-4" aria-label="Breadcrumb">
      <ol className="inline-flex items-center space-x-1 md:space-x-3">
        <li className="inline-flex items-center">
          <Link
            to="/"
            className="text-gray-700 hover:text-gray-900 inline-flex items-center"
          >
            Home
          </Link>
        </li>
        
        {pathnames.map((pathname, index) => {
          const routeTo = `/${pathnames.slice(0, index + 1).join("/")}`;
          const isLast = index === pathnames.length - 1;
          
          return (
            <li key={pathname} className="inline-flex items-center">
              <ChevronRight className="w-4 h-4 text-gray-400 mx-1" />
              {isLast ? (
                <span className="text-gray-500 ml-1 md:ml-2">
                  {pathname.charAt(0).toUpperCase() + pathname.slice(1)}
                </span>
              ) : (
                <Link
                  to={routeTo}
                  className="text-gray-700 hover:text-gray-900 ml-1 md:ml-2"
                >
                  {pathname.charAt(0).toUpperCase() + pathname.slice(1)}
                </Link>
              )}
            </li>
          );
        })}
      </ol>
    </nav>
  );
};
```

## SharePoint Context Management

### Context Provider Pattern

```typescript
// contexts/SharePointContext.tsx
import React, { createContext, useContext, useEffect, useState } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { SPFI } from "@pnp/sp";
import { getSP } from "../services/pnpConfig";

type SharePointContextType = {
  context: WebPartContext;
  sp: SPFI;
  siteUrl: string;
  isTeamsContext: boolean;
};

const SharePointContext = createContext<SharePointContextType | undefined>(undefined);

export const SharePointProvider: React.FC<{
  context: WebPartContext;
  children: React.ReactNode;
}> = ({ context, children }) => {
  const [sp] = useState(() => getSP(context));
  const [isTeamsContext] = useState(() => !!context.sdks.microsoftTeams);

  const value: SharePointContextType = {
    context,
    sp,
    siteUrl: context.pageContext.web.absoluteUrl,
    isTeamsContext
  };

  return (
    <SharePointContext.Provider value={value}>
      {children}
    </SharePointContext.Provider>
  );
};

export const useSharePoint = () => {
  const context = useContext(SharePointContext);
  if (!context) {
    throw new Error("useSharePoint must be used within SharePointProvider");
  }
  return context;
};
```

### Theme Integration

```typescript
import React, { useEffect } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";

export const useSharePointTheme = (context: WebPartContext) => {
  useEffect(() => {
    const updateCSSVariables = () => {
      const theme = context.domElement.style;
      const root = document.documentElement;
      
      // Map SharePoint theme to CSS variables for Tailwind
      root.style.setProperty('--sp-primary', theme.getPropertyValue('--themePrimary'));
      root.style.setProperty('--sp-secondary', theme.getPropertyValue('--themeSecondary'));
      root.style.setProperty('--sp-tertiary', theme.getPropertyValue('--themeTertiary'));
      root.style.setProperty('--sp-dark', theme.getPropertyValue('--themeDark'));
      root.style.setProperty('--sp-darker', theme.getPropertyValue('--themeDarker'));
      root.style.setProperty('--sp-darkest', theme.getPropertyValue('--themeDarkest'));
      root.style.setProperty('--sp-light', theme.getPropertyValue('--themeLight'));
      root.style.setProperty('--sp-lighter', theme.getPropertyValue('--themeLighter'));
      root.style.setProperty('--sp-lightest', theme.getPropertyValue('--themeLightest'));
    };

    updateCSSVariables();
    
    // Listen for theme changes
    const observer = new MutationObserver(updateCSSVariables);
    observer.observe(context.domElement, { 
      attributes: true, 
      attributeFilter: ['style'] 
    });

    return () => observer.disconnect();
  }, [context]);
};
```

## Enhanced Error Handling and Loading States

### Global Error Handler Hook

```typescript
import { useState, useCallback } from "react";
import { toast } from "@/components/ui/toast";

type ErrorType = "permission" | "network" | "validation" | "unknown";

type SharePointError = {
  message: string;
  type: ErrorType;
  statusCode?: number;
  details?: any;
};

export const useErrorHandler = () => {
  const [errors, setErrors] = useState<SharePointError[]>([]);

  const handleError = useCallback((error: any, context?: string) => {
    let errorType: ErrorType = "unknown";
    let message = "An unexpected error occurred";
    let statusCode: number | undefined;

    if (error?.status === 403) {
      errorType = "permission";
      message = "You don't have permission to perform this action";
      statusCode = 403;
    } else if (error?.status === 404) {
      errorType = "network";
      message = "The requested resource was not found";
      statusCode = 404;
    } else if (error?.message?.includes("validation")) {
      errorType = "validation";
      message = error.message;
    } else if (error?.message) {
      message = error.message;
    }

    const spError: SharePointError = {
      message: context ? `${context}: ${message}` : message,
      type: errorType,
      statusCode,
      details: error
    };

    setErrors(prev => [...prev, spError]);

    // Show toast notification
    toast({
      title: errorType === "permission" ? "Access Denied" : "Error",
      description: spError.message,
      variant: "destructive"
    });

    console.error("SharePoint Error:", spError);
    return spError;
  }, []);

  const clearErrors = useCallback(() => {
    setErrors([]);
  }, []);

  return { errors, handleError, clearErrors };
};
```

### Loading State Manager

```typescript
import { useState, useCallback, useRef } from "react";

type LoadingState = {
  [key: string]: boolean;
};

export const useLoadingState = () => {
  const [loadingStates, setLoadingStates] = useState<LoadingState>({});
  const timeoutRefs = useRef<{ [key: string]: NodeJS.Timeout }>({});

  const setLoading = useCallback((key: string, isLoading: boolean, minDuration?: number) => {
    if (isLoading && minDuration) {
      // Set loading immediately
      setLoadingStates(prev => ({ ...prev, [key]: true }));
      
      // Clear any existing timeout
      if (timeoutRefs.current[key]) {
        clearTimeout(timeoutRefs.current[key]);
      }
      
      // Set minimum duration
      timeoutRefs.current[key] = setTimeout(() => {
        setLoadingStates(prev => ({ ...prev, [key]: false }));
        delete timeoutRefs.current[key];
      }, minDuration);
    } else if (!isLoading && timeoutRefs.current[key]) {
      // Don't clear loading if minimum duration hasn't passed
      return;
    } else {
      setLoadingStates(prev => ({ ...prev, [key]: isLoading }));
    }
  }, []);

  const isLoading = useCallback((key: string) => {
    return loadingStates[key] || false;
  }, [loadingStates]);

  const isAnyLoading = useCallback(() => {
    return Object.values(loadingStates).some(state => state);
  }, [loadingStates]);

  return { setLoading, isLoading, isAnyLoading };
};
```

### Retry Logic Component

```typescript
import React, { useState, useCallback } from "react";
import { Button } from "@/components/ui/button";
import { RefreshCw } from "lucide-react";

type RetryConfig = {
  maxRetries?: number;
  retryDelay?: number;
  backoffMultiplier?: number;
};

type RetryWrapperProps = {
  onRetry: () => Promise<void>;
  error: Error | null;
  children: React.ReactNode;
  config?: RetryConfig;
};

export const RetryWrapper: React.FC<RetryWrapperProps> = ({
  onRetry,
  error,
  children,
  config = {}
}) => {
  const {
    maxRetries = 3,
    retryDelay = 1000,
    backoffMultiplier = 2
  } = config;

  const [retryCount, setRetryCount] = useState(0);
  const [isRetrying, setIsRetrying] = useState(false);

  const handleRetry = useCallback(async () => {
    if (retryCount >= maxRetries) {
      return;
    }

    setIsRetrying(true);
    
    // Calculate delay with exponential backoff
    const delay = retryDelay * Math.pow(backoffMultiplier, retryCount);
    
    await new Promise(resolve => setTimeout(resolve, delay));
    
    try {
      await onRetry();
      setRetryCount(0); // Reset on success
    } catch (err) {
      setRetryCount(prev => prev + 1);
    } finally {
      setIsRetrying(false);
    }
  }, [onRetry, retryCount, maxRetries, retryDelay, backoffMultiplier]);

  if (error) {
    return (
      <div className="flex flex-col items-center justify-center p-8 space-y-4">
        <div className="text-center">
          <p className="text-red-600 font-medium">Something went wrong</p>
          <p className="text-sm text-gray-600 mt-1">{error.message}</p>
        </div>
        
        {retryCount < maxRetries && (
          <Button
            onClick={handleRetry}
            disabled={isRetrying}
            variant="outline"
            size="sm"
          >
            {isRetrying ? (
              <>
                <RefreshCw className="mr-2 h-4 w-4 animate-spin" />
                Retrying...
              </>
            ) : (
              <>
                <RefreshCw className="mr-2 h-4 w-4" />
                Retry ({maxRetries - retryCount} attempts left)
              </>
            )}
          </Button>
        )}
        
        {retryCount >= maxRetries && (
          <p className="text-sm text-gray-500">
            Maximum retry attempts reached. Please refresh the page.
          </p>
        )}
      </div>
    );
  }

  return <>{children}</>;
};
```

### Skeleton Loading Components

```typescript
import React from "react";
import { Skeleton } from "@/components/ui/skeleton";

export const ListItemSkeleton: React.FC = () => (
  <div className="p-4 bg-white border rounded-lg">
    <Skeleton className="h-5 w-3/4 mb-2" />
    <Skeleton className="h-4 w-1/2 mb-3" />
    <div className="flex gap-2">
      <Skeleton className="h-6 w-20 rounded-full" />
      <Skeleton className="h-6 w-24 rounded-full" />
    </div>
  </div>
);

export const TableSkeleton: React.FC<{ rows?: number }> = ({ rows = 5 }) => (
  <div className="w-full">
    <div className="border rounded-lg">
      <div className="border-b bg-gray-50 p-4">
        <div className="flex gap-4">
          <Skeleton className="h-4 w-32" />
          <Skeleton className="h-4 w-24" />
          <Skeleton className="h-4 w-28" />
          <Skeleton className="h-4 w-20" />
        </div>
      </div>
      {[...Array(rows)].map((_, i) => (
        <div key={i} className="border-b p-4">
          <div className="flex gap-4">
            <Skeleton className="h-4 w-48" />
            <Skeleton className="h-4 w-16" />
            <Skeleton className="h-4 w-20" />
            <Skeleton className="h-4 w-24" />
          </div>
        </div>
      ))}
    </div>
  </div>
);

export const FormSkeleton: React.FC = () => (
  <div className="space-y-6">
    <div>
      <Skeleton className="h-4 w-20 mb-2" />
      <Skeleton className="h-10 w-full" />
    </div>
    <div>
      <Skeleton className="h-4 w-24 mb-2" />
      <Skeleton className="h-20 w-full" />
    </div>
    <div className="flex gap-4">
      <Skeleton className="h-10 w-24" />
      <Skeleton className="h-10 w-24" />
    </div>
  </div>
);
```

### Optimistic Updates Pattern

```typescript
import { useState, useCallback } from "react";
import { useSharePointList } from "./useSharePointList";

type OptimisticUpdate<T> = {
  id: string;
  type: "create" | "update" | "delete";
  data: T;
  previousData?: T;
};

export const useOptimisticUpdates = <T extends { Id: number }>(
  listName: string,
  context: WebPartContext
) => {
  const { items, loading, error, create, update, delete: deleteItem } = useSharePointList<T>({
    listName,
    context,
    autoLoad: true
  });

  const [optimisticUpdates, setOptimisticUpdates] = useState<OptimisticUpdate<T>[]>([]);
  const [pendingIds, setPendingIds] = useState<Set<string>>(new Set());

  const optimisticCreate = useCallback(async (data: Partial<T>) => {
    const tempId = `temp_${Date.now()}`;
    const optimisticItem = { ...data, Id: -1 } as T;
    
    // Add optimistic update
    setOptimisticUpdates(prev => [...prev, {
      id: tempId,
      type: "create",
      data: optimisticItem
    }]);
    setPendingIds(prev => new Set(prev).add(tempId));

    try {
      const createdItem = await create(data);
      
      // Remove optimistic update and replace with real data
      setOptimisticUpdates(prev => prev.filter(u => u.id !== tempId));
      setPendingIds(prev => {
        const next = new Set(prev);
        next.delete(tempId);
        return next;
      });
      
      return createdItem;
    } catch (error) {
      // Revert optimistic update on error
      setOptimisticUpdates(prev => prev.filter(u => u.id !== tempId));
      setPendingIds(prev => {
        const next = new Set(prev);
        next.delete(tempId);
        return next;
      });
      throw error;
    }
  }, [create]);

  const optimisticUpdate = useCallback(async (id: number, data: Partial<T>) => {
    const updateId = `update_${id}`;
    const currentItem = items.find(item => item.Id === id);
    
    if (!currentItem) return;

    // Add optimistic update
    setOptimisticUpdates(prev => [...prev, {
      id: updateId,
      type: "update",
      data: { ...currentItem, ...data },
      previousData: currentItem
    }]);
    setPendingIds(prev => new Set(prev).add(updateId));

    try {
      const updatedItem = await update(id, data);
      
      // Remove optimistic update
      setOptimisticUpdates(prev => prev.filter(u => u.id !== updateId));
      setPendingIds(prev => {
        const next = new Set(prev);
        next.delete(updateId);
        return next;
      });
      
      return updatedItem;
    } catch (error) {
      // Revert optimistic update on error
      setOptimisticUpdates(prev => prev.filter(u => u.id !== updateId));
      setPendingIds(prev => {
        const next = new Set(prev);
        next.delete(updateId);
        return next;
      });
      throw error;
    }
  }, [items, update]);

  // Compute optimistic items
  const optimisticItems = useCallback(() => {
    let result = [...items];

    for (const update of optimisticUpdates) {
      switch (update.type) {
        case "create":
          result.push(update.data);
          break;
        case "update":
          result = result.map(item => 
            item.Id === update.data.Id ? update.data : item
          );
          break;
        case "delete":
          result = result.filter(item => item.Id !== update.data.Id);
          break;
      }
    }

    return result;
  }, [items, optimisticUpdates]);

  return {
    items: optimisticItems(),
    loading,
    error,
    create: optimisticCreate,
    update: optimisticUpdate,
    delete: deleteItem,
    isPending: (id: number | string) => pendingIds.has(String(id))
  };
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

### 5. Handle SharePoint Permissions

```typescript
// ✅ Good - Check permissions before operations
export const SecureComponent: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const [hasPermission, setHasPermission] = useState(false);
  const [checkingPermission, setCheckingPermission] = useState(true);

  useEffect(() => {
    const checkPermissions = async () => {
      try {
        const sp = getSP(context);
        const perms = await sp.web.currentUserHasPermissions(
          PermissionKind.AddListItems
        );
        setHasPermission(perms);
      } catch (error) {
        console.error("Permission check failed:", error);
        setHasPermission(false);
      } finally {
        setCheckingPermission(false);
      }
    };

    checkPermissions();
  }, [context]);

  if (checkingPermission) {
    return <Skeleton className="h-32 w-full" />;
  }

  if (!hasPermission) {
    return (
      <div className="p-4 bg-yellow-50 border border-yellow-200 rounded-md">
        <p className="text-yellow-800">
          You don't have permission to perform this action.
        </p>
      </div>
    );
  }

  return <div>Secure content here</div>;
};
```

### 6. Optimize SharePoint API Calls

```typescript
// ✅ Good - Batch operations and caching
const useCachedSharePointData = <T>(key: string, fetcher: () => Promise<T>) => {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const cacheKey = `sp_cache_${key}`;

  useEffect(() => {
    const loadData = async () => {
      // Check session storage cache first
      const cached = sessionStorage.getItem(cacheKey);
      if (cached) {
        const { data, timestamp } = JSON.parse(cached);
        const cacheAge = Date.now() - timestamp;
        
        // Use cache if less than 5 minutes old
        if (cacheAge < 5 * 60 * 1000) {
          setData(data);
          setLoading(false);
          return;
        }
      }

      // Fetch fresh data
      try {
        const freshData = await fetcher();
        setData(freshData);
        
        // Cache the data
        sessionStorage.setItem(cacheKey, JSON.stringify({
          data: freshData,
          timestamp: Date.now()
        }));
      } catch (error) {
        console.error("Failed to fetch data:", error);
      } finally {
        setLoading(false);
      }
    };

    loadData();
  }, [key, cacheKey]);

  return { data, loading };
};
```

## Next Steps

- [Component Architecture](./05-component-architecture.md)
- [Routing Setup](./06-routing-setup.md)
- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
