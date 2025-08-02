# TypeScript Configuration & Patterns

This guide covers TypeScript setup and coding patterns for SharePoint Framework development, emphasizing type definitions over interfaces and functional programming patterns.

## TypeScript Configuration

### Strict TypeScript Setup

Ensure your `tsconfig.json` includes strict type checking:

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

## Type Definition Patterns

### Using `type` over `interface`

Following your preference, use `type` definitions instead of `interface`:

```typescript
// ✅ Preferred: Type definitions
type UserData = {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user" | "guest";
  createdAt: Date;
  isActive: boolean;
};

type ApiResponse<T> = {
  data: T;
  success: boolean;
  message?: string;
  errors?: string[];
};

// ✅ Union types for better type safety
type Status = "loading" | "success" | "error" | "idle";

type ComponentState = {
  status: Status;
  data: UserData[] | null;
  error: string | null;
};
```

### Complex Type Compositions

```typescript
// Base types
type BaseEntity = {
  id: string;
  createdAt: Date;
  updatedAt: Date;
};

type SharePointItem = BaseEntity & {
  Title: string;
  Author: {
    Title: string;
    Email: string;
  };
};

// Extend for specific list items
type ProjectItem = SharePointItem & {
  ProjectName: string;
  StartDate: Date;
  EndDate: Date;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  TeamMembers: string[];
};

type TaskItem = SharePointItem & {
  TaskName: string;
  ProjectId: string;
  AssignedTo: string;
  Priority: "Low" | "Medium" | "High" | "Critical";
  DueDate: Date;
  IsCompleted: boolean;
};
```

### Utility Types for SharePoint

```typescript
// SharePoint field types
type SPFieldValue<T> = T | null;

type SPUser = {
  Title: string;
  Email: string;
  Id: number;
};

type SPLookupValue = {
  Id: number;
  Title: string;
};

// Generic SharePoint list item
type SPListItem<T = Record<string, unknown>> = {
  Id: number;
  Title: string;
  Created: string;
  Modified: string;
  Author: SPUser;
  Editor: SPUser;
} & T;

// For list operations
type ListItemCreateData<T> = Omit<
  T,
  "Id" | "Created" | "Modified" | "Author" | "Editor"
>;
type ListItemUpdateData<T> = Partial<ListItemCreateData<T>>;
```

## Functional Component Types

### Component Props with Generic Constraints

```typescript
// Base props for all components
type BaseComponentProps = {
  className?: string;
  children?: React.ReactNode;
};

// Props for data display components
type DataComponentProps<T> = BaseComponentProps & {
  data: T[];
  loading?: boolean;
  error?: string;
  onRefresh?: () => void;
};

// Props with event handlers
type FormComponentProps<T> = BaseComponentProps & {
  initialData?: T;
  onSubmit: (data: T) => Promise<void>;
  onCancel?: () => void;
  validationSchema?: Record<keyof T, (value: unknown) => string | null>;
};

// Example usage
type ProjectListProps = DataComponentProps<ProjectItem> & {
  onProjectSelect: (project: ProjectItem) => void;
  selectedProjectId?: string;
};

const ProjectList: React.FC<ProjectListProps> = ({
  data,
  loading,
  error,
  onProjectSelect,
  selectedProjectId,
  className,
}) => {
  // Component implementation
};
```

### Hook Return Types

```typescript
// Custom hook return types
type UseSharePointListResult<T> = {
  items: T[];
  loading: boolean;
  error: string | null;
  refetch: () => Promise<void>;
  create: (item: ListItemCreateData<T>) => Promise<T>;
  update: (id: number, item: ListItemUpdateData<T>) => Promise<T>;
  delete: (id: number) => Promise<void>;
};

type UseFormState<T> = {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isValid: boolean;
  isSubmitting: boolean;
  setValue: <K extends keyof T>(field: K, value: T[K]) => void;
  setError: <K extends keyof T>(field: K, error: string) => void;
  submit: () => Promise<void>;
  reset: () => void;
};
```

## Advanced Type Patterns

### Conditional Types for API Responses

```typescript
// Conditional types for different response states
type AsyncData<T> =
  | { status: "loading"; data: null; error: null }
  | { status: "success"; data: T; error: null }
  | { status: "error"; data: null; error: string };

// Discriminated unions for component states
type ComponentMode =
  | { mode: "view"; selectedItem: ProjectItem }
  | { mode: "edit"; selectedItem: ProjectItem }
  | { mode: "create"; selectedItem: null }
  | { mode: "list"; selectedItem: null };
```

### Template Literal Types

```typescript
// For SharePoint column names
type ColumnPrefix = "OData_" | "odata_";
type ColumnSuffix = "_x0020_" | "Value";

type SPColumnName<T extends string> = `${ColumnPrefix}${T}${ColumnSuffix}`;

// For CSS class generation
type TailwindSize = "xs" | "sm" | "md" | "lg" | "xl";
type TailwindVariant = "primary" | "secondary" | "destructive" | "outline";

type ButtonClass = `btn-${TailwindVariant}-${TailwindSize}`;
```

### Mapped Types for Form Validation

```typescript
// Create validation rules from type
type ValidationRules<T> = {
  [K in keyof T]?: {
    required?: boolean;
    minLength?: number;
    maxLength?: number;
    pattern?: RegExp;
    custom?: (value: T[K]) => string | null;
  };
};

// Example usage
const projectValidation: ValidationRules<ProjectItem> = {
  ProjectName: {
    required: true,
    minLength: 3,
    maxLength: 100,
  },
  StartDate: {
    required: true,
    custom: (date) => {
      if (date < new Date()) {
        return "Start date cannot be in the past";
      }
      return null;
    },
  },
};
```

## Type Guards and Narrowing

### Custom Type Guards

```typescript
// Type guards for runtime type checking
const isSharePointItem = (item: unknown): item is SharePointItem => {
  return (
    typeof item === "object" &&
    item !== null &&
    "id" in item &&
    "Title" in item &&
    "Author" in item
  );
};

const isProjectItem = (item: SharePointItem): item is ProjectItem => {
  return "ProjectName" in item && "StartDate" in item;
};

// Usage in components
const processItem = (item: unknown) => {
  if (isSharePointItem(item) && isProjectItem(item)) {
    // TypeScript now knows item is ProjectItem
    console.log(item.ProjectName);
  }
};
```

### Assertion Functions

```typescript
// Custom assertion functions
const assertIsValidEmail = (value: string): asserts value is string => {
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
  if (!emailRegex.test(value)) {
    throw new Error("Invalid email format");
  }
};

const assertIsProjectItem = (
  item: SharePointItem
): asserts item is ProjectItem => {
  if (!("ProjectName" in item)) {
    throw new Error("Item is not a valid ProjectItem");
  }
};
```

## Generic Utility Functions

### Type-safe API helpers

```typescript
// Generic SharePoint list service
type SPListService<T extends SharePointItem> = {
  listName: string;
  getAll: () => Promise<T[]>;
  getById: (id: number) => Promise<T>;
  create: (item: ListItemCreateData<T>) => Promise<T>;
  update: (id: number, item: ListItemUpdateData<T>) => Promise<T>;
  delete: (id: number) => Promise<void>;
};

// Factory function with proper typing
const createListService = <T extends SharePointItem>(
  listName: string,
  context: WebPartContext
): SPListService<T> => {
  return {
    listName,
    getAll: async () => {
      // Implementation
      return [] as T[];
    },
    getById: async (id: number) => {
      // Implementation
      return {} as T;
    },
    create: async (item: ListItemCreateData<T>) => {
      // Implementation
      return {} as T;
    },
    update: async (id: number, item: ListItemUpdateData<T>) => {
      // Implementation
      return {} as T;
    },
    delete: async (id: number) => {
      // Implementation
    },
  };
};
```

## Best Practices

### 1. Prefer Type Definitions

```typescript
// ✅ Good
type UserPreferences = {
  theme: "light" | "dark";
  language: string;
};

// ❌ Avoid (as per your preference)
interface UserPreferences {
  theme: "light" | "dark";
  language: string;
}
```

### 2. Use Const Assertions

```typescript
// ✅ Good - creates literal type
const statuses = ["active", "inactive", "pending"] as const;
type Status = (typeof statuses)[number]; // 'active' | 'inactive' | 'pending'

// ✅ Good - object with literal types
const theme = {
  colors: {
    primary: "#0066cc",
    secondary: "#6c757d",
  },
} as const;
```

### 3. Discriminated Unions

```typescript
// ✅ Good - discriminated union
type LoadingState =
  | { type: "idle" }
  | { type: "loading" }
  | { type: "success"; data: unknown }
  | { type: "error"; error: string };

// Usage with type narrowing
const handleState = (state: LoadingState) => {
  switch (state.type) {
    case "success":
      // TypeScript knows state.data exists
      return state.data;
    case "error":
      // TypeScript knows state.error exists
      return state.error;
    default:
      return null;
  }
};
```

### 4. Avoid `any` - Use `unknown`

```typescript
// ✅ Good
const processData = (data: unknown) => {
  if (typeof data === "object" && data !== null) {
    // Type guard needed
    return data;
  }
  throw new Error("Invalid data");
};

// ❌ Avoid
const processData = (data: any) => {
  return data.someProperty; // No type safety
};
```

## Next Steps

- [React Functional Patterns](./04-react-functional-patterns.md)
- [Component Architecture](./05-component-architecture.md)
- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
