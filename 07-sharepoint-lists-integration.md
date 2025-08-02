# SharePoint Lists Integration

This guide covers integrating SharePoint Lists as your data layer, including CRUD operations, relationships, and advanced querying patterns.

## SharePoint List Setup

### Creating Lists via Code

```typescript
// src/lib/listSetup.ts
import { WebPartContext } from "@microsoft/sp-webpart-base";

type ListFieldDefinition = {
  name: string;
  type:
    | "Text"
    | "Number"
    | "Boolean"
    | "DateTime"
    | "Choice"
    | "Lookup"
    | "User";
  required?: boolean;
  choices?: string[];
  lookupList?: string;
  lookupField?: string;
};

type ListDefinition = {
  title: string;
  description: string;
  template: number; // 100 for Generic List
  fields: ListFieldDefinition[];
};

export const createListIfNotExists = async (
  context: WebPartContext,
  listDef: ListDefinition
): Promise<void> => {
  try {
    // Check if list exists
    const web = context.pageContext.web;
    const lists = web.lists;

    try {
      await lists.getByTitle(listDef.title).get();
      console.log(`List '${listDef.title}' already exists`);
      return;
    } catch {
      // List doesn't exist, create it
    }

    // Create the list
    const listCreateInfo = {
      Title: listDef.title,
      Description: listDef.description,
      BaseTemplate: listDef.template,
    };

    const list = await lists.add(listCreateInfo);
    console.log(`Created list '${listDef.title}'`);

    // Add custom fields
    for (const field of listDef.fields) {
      await addFieldToList(context, listDef.title, field);
    }
  } catch (error) {
    console.error(`Error creating list '${listDef.title}':`, error);
  }
};

const addFieldToList = async (
  context: WebPartContext,
  listTitle: string,
  field: ListFieldDefinition
): Promise<void> => {
  const list = context.pageContext.web.lists.getByTitle(listTitle);

  switch (field.type) {
    case "Text":
      await list.fields.addText(field.name, { Required: field.required });
      break;
    case "Number":
      await list.fields.addNumber(field.name, { Required: field.required });
      break;
    case "Boolean":
      await list.fields.addBoolean(field.name, { Required: field.required });
      break;
    case "DateTime":
      await list.fields.addDateTime(field.name, {
        Required: field.required,
        DateTimeCalendarType: 0, // Gregorian
        Format: 1, // Date and Time
      });
      break;
    case "Choice":
      await list.fields.addChoice(field.name, {
        Required: field.required,
        Choices: field.choices || [],
      });
      break;
    case "User":
      await list.fields.addUser(field.name, { Required: field.required });
      break;
  }
};

// Example list definitions
export const projectListDefinition: ListDefinition = {
  title: "Projects",
  description: "Project management list",
  template: 100,
  fields: [
    { name: "ProjectName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    {
      name: "Status",
      type: "Choice",
      choices: ["Planning", "In Progress", "Completed", "On Hold"],
    },
    { name: "StartDate", type: "DateTime", required: true },
    { name: "EndDate", type: "DateTime" },
    { name: "ProjectManager", type: "User" },
    { name: "Budget", type: "Number" },
    { name: "IsActive", type: "Boolean" },
  ],
};

export const taskListDefinition: ListDefinition = {
  title: "Tasks",
  description: "Task management list",
  template: 100,
  fields: [
    { name: "TaskName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    {
      name: "Priority",
      type: "Choice",
      choices: ["Low", "Medium", "High", "Critical"],
    },
    { name: "DueDate", type: "DateTime" },
    { name: "AssignedTo", type: "User" },
    { name: "EstimatedHours", type: "Number" },
    { name: "ActualHours", type: "Number" },
    { name: "IsCompleted", type: "Boolean" },
  ],
};
```

## Type Definitions for SharePoint Lists

```typescript
// src/types/sharepoint.ts

// Base SharePoint item properties
export type SharePointBaseItem = {
  Id: number;
  Title: string;
  Created: string;
  Modified: string;
  Author: {
    Title: string;
    Email: string;
    Id: number;
  };
  Editor: {
    Title: string;
    Email: string;
    Id: number;
  };
};

// SharePoint field types
export type SPUser = {
  Title: string;
  Email: string;
  Id: number;
};

export type SPLookupValue = {
  Id: number;
  Title: string;
};

export type SPChoice = string;

// Project list item type
export type ProjectItem = SharePointBaseItem & {
  ProjectName: string;
  Description: string;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  StartDate: string;
  EndDate: string;
  ProjectManager: SPUser;
  Budget: number;
  IsActive: boolean;
};

// Task list item type
export type TaskItem = SharePointBaseItem & {
  TaskName: string;
  Description: string;
  Priority: "Low" | "Medium" | "High" | "Critical";
  DueDate: string;
  AssignedTo: SPUser;
  EstimatedHours: number;
  ActualHours: number;
  IsCompleted: boolean;
  ProjectId?: SPLookupValue; // If linking to projects
};

// For creating/updating items (omit readonly fields)
export type ProjectItemCreate = Omit<
  ProjectItem,
  "Id" | "Created" | "Modified" | "Author" | "Editor"
>;

export type ProjectItemUpdate = Partial<ProjectItemCreate>;

export type TaskItemCreate = Omit<
  TaskItem,
  "Id" | "Created" | "Modified" | "Author" | "Editor"
>;

export type TaskItemUpdate = Partial<TaskItemCreate>;
```

## SharePoint API Service Layer

```typescript
// src/lib/sharePointService.ts
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { sp } from "@pnp/sp/presets/all";

export class SharePointService {
  private context: WebPartContext;

  constructor(context: WebPartContext) {
    this.context = context;

    // Initialize PnPjs
    sp.setup({
      spfxContext: context as any,
    });
  }

  // Generic list operations
  async getListItems<T>(
    listName: string,
    select?: string[],
    expand?: string[],
    filter?: string,
    orderBy?: string,
    top?: number
  ): Promise<T[]> {
    try {
      let query = sp.web.lists.getByTitle(listName).items;

      if (select && select.length > 0) {
        query = query.select(...select);
      }

      if (expand && expand.length > 0) {
        query = query.expand(...expand);
      }

      if (filter) {
        query = query.filter(filter);
      }

      if (orderBy) {
        query = query.orderBy(orderBy);
      }

      if (top) {
        query = query.top(top);
      }

      const items = await query.get();
      return items as T[];
    } catch (error) {
      console.error(`Error fetching items from ${listName}:`, error);
      throw error;
    }
  }

  async getListItemById<T>(listName: string, id: number): Promise<T> {
    try {
      const item = await sp.web.lists
        .getByTitle(listName)
        .items.getById(id)
        .get();
      return item as T;
    } catch (error) {
      console.error(`Error fetching item ${id} from ${listName}:`, error);
      throw error;
    }
  }

  async createListItem<T>(listName: string, item: Partial<T>): Promise<T> {
    try {
      const result = await sp.web.lists.getByTitle(listName).items.add(item);
      return result.data as T;
    } catch (error) {
      console.error(`Error creating item in ${listName}:`, error);
      throw error;
    }
  }

  async updateListItem<T>(
    listName: string,
    id: number,
    updates: Partial<T>
  ): Promise<T> {
    try {
      await sp.web.lists.getByTitle(listName).items.getById(id).update(updates);

      // Return updated item
      return this.getListItemById<T>(listName, id);
    } catch (error) {
      console.error(`Error updating item ${id} in ${listName}:`, error);
      throw error;
    }
  }

  async deleteListItem(listName: string, id: number): Promise<void> {
    try {
      await sp.web.lists.getByTitle(listName).items.getById(id).delete();
    } catch (error) {
      console.error(`Error deleting item ${id} from ${listName}:`, error);
      throw error;
    }
  }

  // Batch operations
  async batchCreateItems<T>(
    listName: string,
    items: Partial<T>[]
  ): Promise<T[]> {
    try {
      const batch = sp.web.createBatch();
      const promises: Promise<any>[] = [];

      items.forEach((item) => {
        promises.push(
          sp.web.lists.getByTitle(listName).items.inBatch(batch).add(item)
        );
      });

      await batch.execute();
      const results = await Promise.all(promises);
      return results.map((result) => result.data) as T[];
    } catch (error) {
      console.error(`Error batch creating items in ${listName}:`, error);
      throw error;
    }
  }

  // Search functionality
  async searchListItems<T>(
    listName: string,
    searchText: string,
    searchFields: string[] = ["Title"]
  ): Promise<T[]> {
    try {
      const filters = searchFields
        .map((field) => `substringof('${searchText}', ${field})`)
        .join(" or ");

      return this.getListItems<T>(listName, undefined, undefined, filters);
    } catch (error) {
      console.error(`Error searching items in ${listName}:`, error);
      throw error;
    }
  }
}

// Specialized services for each list
export class ProjectService extends SharePointService {
  private listName = "Projects";

  async getAllProjects(): Promise<ProjectItem[]> {
    return this.getListItems<ProjectItem>(
      this.listName,
      ["*", "ProjectManager/Title", "ProjectManager/Email"],
      ["ProjectManager"]
    );
  }

  async getActiveProjects(): Promise<ProjectItem[]> {
    return this.getListItems<ProjectItem>(
      this.listName,
      ["*", "ProjectManager/Title", "ProjectManager/Email"],
      ["ProjectManager"],
      "IsActive eq true"
    );
  }

  async getProjectsByStatus(
    status: ProjectItem["Status"]
  ): Promise<ProjectItem[]> {
    return this.getListItems<ProjectItem>(
      this.listName,
      ["*", "ProjectManager/Title", "ProjectManager/Email"],
      ["ProjectManager"],
      `Status eq '${status}'`
    );
  }

  async createProject(project: ProjectItemCreate): Promise<ProjectItem> {
    return this.createListItem<ProjectItem>(this.listName, project);
  }

  async updateProject(
    id: number,
    updates: ProjectItemUpdate
  ): Promise<ProjectItem> {
    return this.updateListItem<ProjectItem>(this.listName, id, updates);
  }

  async deleteProject(id: number): Promise<void> {
    return this.deleteListItem(this.listName, id);
  }

  async searchProjects(searchText: string): Promise<ProjectItem[]> {
    return this.searchListItems<ProjectItem>(this.listName, searchText, [
      "Title",
      "ProjectName",
      "Description",
    ]);
  }
}

export class TaskService extends SharePointService {
  private listName = "Tasks";

  async getAllTasks(): Promise<TaskItem[]> {
    return this.getListItems<TaskItem>(
      this.listName,
      ["*", "AssignedTo/Title", "AssignedTo/Email"],
      ["AssignedTo"]
    );
  }

  async getTasksByProject(projectId: number): Promise<TaskItem[]> {
    return this.getListItems<TaskItem>(
      this.listName,
      ["*", "AssignedTo/Title", "AssignedTo/Email"],
      ["AssignedTo"],
      `ProjectId eq ${projectId}`
    );
  }

  async getTasksByUser(userId: number): Promise<TaskItem[]> {
    return this.getListItems<TaskItem>(
      this.listName,
      ["*", "AssignedTo/Title", "AssignedTo/Email"],
      ["AssignedTo"],
      `AssignedTo eq ${userId}`
    );
  }

  async getOverdueTasks(): Promise<TaskItem[]> {
    const today = new Date().toISOString();
    return this.getListItems<TaskItem>(
      this.listName,
      ["*", "AssignedTo/Title", "AssignedTo/Email"],
      ["AssignedTo"],
      `DueDate lt datetime'${today}' and IsCompleted eq false`
    );
  }

  async createTask(task: TaskItemCreate): Promise<TaskItem> {
    return this.createListItem<TaskItem>(this.listName, task);
  }

  async updateTask(id: number, updates: TaskItemUpdate): Promise<TaskItem> {
    return this.updateListItem<TaskItem>(this.listName, id, updates);
  }

  async completeTask(id: number): Promise<TaskItem> {
    return this.updateTask(id, { IsCompleted: true });
  }

  async deleteTask(id: number): Promise<void> {
    return this.deleteListItem(this.listName, id);
  }
}
```

## Custom Hooks for SharePoint Lists

```typescript
// src/hooks/useSharePointList.ts
import { useState, useEffect, useCallback } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { SharePointService } from "@/lib/sharePointService";

type UseSharePointListOptions<T> = {
  listName: string;
  context: WebPartContext;
  select?: string[];
  expand?: string[];
  filter?: string;
  orderBy?: string;
  top?: number;
  autoLoad?: boolean;
};

type UseSharePointListResult<T> = {
  items: T[];
  loading: boolean;
  error: string | null;
  refresh: () => Promise<void>;
  create: (item: Partial<T>) => Promise<T>;
  update: (id: number, updates: Partial<T>) => Promise<T>;
  delete: (id: number) => Promise<void>;
  search: (query: string, fields?: string[]) => Promise<T[]>;
};

export const useSharePointList = <T extends { Id: number }>(
  options: UseSharePointListOptions<T>
): UseSharePointListResult<T> => {
  const [items, setItems] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const {
    listName,
    context,
    select,
    expand,
    filter,
    orderBy,
    top,
    autoLoad = true,
  } = options;

  const service = new SharePointService(context);

  const loadItems = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);

      const data = await service.getListItems<T>(
        listName,
        select,
        expand,
        filter,
        orderBy,
        top
      );

      setItems(data);
    } catch (err) {
      const message =
        err instanceof Error ? err.message : "Failed to load items";
      setError(message);
    } finally {
      setLoading(false);
    }
  }, [listName, select, expand, filter, orderBy, top, service]);

  const createItem = useCallback(
    async (item: Partial<T>): Promise<T> => {
      try {
        const newItem = await service.createListItem<T>(listName, item);
        setItems((prev) => [...prev, newItem]);
        return newItem;
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to create item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, service]
  );

  const updateItem = useCallback(
    async (id: number, updates: Partial<T>): Promise<T> => {
      try {
        const updatedItem = await service.updateListItem<T>(
          listName,
          id,
          updates
        );
        setItems((prev) =>
          prev.map((item) => (item.Id === id ? updatedItem : item))
        );
        return updatedItem;
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to update item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, service]
  );

  const deleteItem = useCallback(
    async (id: number): Promise<void> => {
      try {
        await service.deleteListItem(listName, id);
        setItems((prev) => prev.filter((item) => item.Id !== id));
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to delete item";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, service]
  );

  const searchItems = useCallback(
    async (query: string, fields?: string[]): Promise<T[]> => {
      try {
        return await service.searchListItems<T>(listName, query, fields);
      } catch (err) {
        const message =
          err instanceof Error ? err.message : "Failed to search items";
        setError(message);
        throw new Error(message);
      }
    },
    [listName, service]
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
    search: searchItems,
  };
};

// Specialized hooks
export const useProjects = (context: WebPartContext) => {
  return useSharePointList<ProjectItem>({
    listName: "Projects",
    context,
    select: ["*", "ProjectManager/Title", "ProjectManager/Email"],
    expand: ["ProjectManager"],
    orderBy: "Created desc",
  });
};

export const useTasks = (context: WebPartContext, projectId?: number) => {
  return useSharePointList<TaskItem>({
    listName: "Tasks",
    context,
    select: ["*", "AssignedTo/Title", "AssignedTo/Email"],
    expand: ["AssignedTo"],
    filter: projectId ? `ProjectId eq ${projectId}` : undefined,
    orderBy: "DueDate asc",
  });
};
```

## Advanced Query Patterns

### Complex Filtering and Sorting

```typescript
// src/lib/sharePointQueries.ts
import { SharePointService } from "./sharePointService";

export class SharePointQueryBuilder {
  private service: SharePointService;

  constructor(service: SharePointService) {
    this.service = service;
  }

  // Advanced project queries
  async getProjectsWithFilters(filters: {
    status?: string[];
    startDateFrom?: Date;
    startDateTo?: Date;
    managerId?: number;
    budgetMin?: number;
    budgetMax?: number;
  }): Promise<ProjectItem[]> {
    const filterParts: string[] = [];

    if (filters.status && filters.status.length > 0) {
      const statusFilter = filters.status
        .map((status) => `Status eq '${status}'`)
        .join(" or ");
      filterParts.push(`(${statusFilter})`);
    }

    if (filters.startDateFrom) {
      const dateStr = filters.startDateFrom.toISOString();
      filterParts.push(`StartDate ge datetime'${dateStr}'`);
    }

    if (filters.startDateTo) {
      const dateStr = filters.startDateTo.toISOString();
      filterParts.push(`StartDate le datetime'${dateStr}'`);
    }

    if (filters.managerId) {
      filterParts.push(`ProjectManager eq ${filters.managerId}`);
    }

    if (filters.budgetMin !== undefined) {
      filterParts.push(`Budget ge ${filters.budgetMin}`);
    }

    if (filters.budgetMax !== undefined) {
      filterParts.push(`Budget le ${filters.budgetMax}`);
    }

    const filter = filterParts.join(" and ");

    return this.service.getListItems<ProjectItem>(
      "Projects",
      ["*", "ProjectManager/Title", "ProjectManager/Email"],
      ["ProjectManager"],
      filter,
      "StartDate desc"
    );
  }

  // Get projects with task counts
  async getProjectsWithTaskCounts(): Promise<
    (ProjectItem & { TaskCount: number })[]
  > {
    const projects = await this.service.getListItems<ProjectItem>(
      "Projects",
      ["*", "ProjectManager/Title", "ProjectManager/Email"],
      ["ProjectManager"]
    );

    const projectsWithCounts = await Promise.all(
      projects.map(async (project) => {
        const tasks = await this.service.getListItems<TaskItem>(
          "Tasks",
          ["Id"],
          undefined,
          `ProjectId eq ${project.Id}`
        );

        return {
          ...project,
          TaskCount: tasks.length,
        };
      })
    );

    return projectsWithCounts;
  }

  // Get user workload (tasks assigned)
  async getUserWorkload(userId: number): Promise<{
    totalTasks: number;
    completedTasks: number;
    pendingTasks: number;
    overdueTasks: number;
    totalEstimatedHours: number;
    totalActualHours: number;
  }> {
    const allTasks = await this.service.getListItems<TaskItem>(
      "Tasks",
      ["*"],
      undefined,
      `AssignedTo eq ${userId}`
    );

    const now = new Date();
    const completedTasks = allTasks.filter((task) => task.IsCompleted);
    const pendingTasks = allTasks.filter((task) => !task.IsCompleted);
    const overdueTasks = allTasks.filter(
      (task) =>
        !task.IsCompleted && task.DueDate && new Date(task.DueDate) < now
    );

    return {
      totalTasks: allTasks.length,
      completedTasks: completedTasks.length,
      pendingTasks: pendingTasks.length,
      overdueTasks: overdueTasks.length,
      totalEstimatedHours: allTasks.reduce(
        (sum, task) => sum + (task.EstimatedHours || 0),
        0
      ),
      totalActualHours: allTasks.reduce(
        (sum, task) => sum + (task.ActualHours || 0),
        0
      ),
    };
  }
}
```

### Relationship Management

```typescript
// src/lib/sharePointRelationships.ts
import { SharePointService } from "./sharePointService";

export class SharePointRelationshipManager {
  private service: SharePointService;

  constructor(service: SharePointService) {
    this.service = service;
  }

  // Create lookup relationships
  async createProjectLookupField(): Promise<void> {
    try {
      const tasksListId = await this.getListId("Tasks");
      const projectsListId = await this.getListId("Projects");

      // Add lookup field to Tasks list
      await this.service.context.pageContext.web.lists
        .getById(tasksListId)
        .fields.addLookup("ProjectLookup", {
          LookupListId: projectsListId,
          LookupFieldName: "Title",
          Required: false,
          RelationshipBehavior: 0, // None
        });

      console.log("Project lookup field created in Tasks list");
    } catch (error) {
      console.error("Error creating lookup field:", error);
    }
  }

  private async getListId(listTitle: string): Promise<string> {
    const list = await this.service.context.pageContext.web.lists
      .getByTitle(listTitle)
      .select("Id")
      .get();
    return list.Id;
  }

  // Get related data
  async getProjectWithTasks(projectId: number): Promise<{
    project: ProjectItem;
    tasks: TaskItem[];
  }> {
    const [project, tasks] = await Promise.all([
      this.service.getListItemById<ProjectItem>("Projects", projectId),
      this.service.getListItems<TaskItem>(
        "Tasks",
        ["*", "AssignedTo/Title", "AssignedTo/Email"],
        ["AssignedTo"],
        `ProjectId eq ${projectId}`
      ),
    ]);

    return { project, tasks };
  }

  // Cascade operations
  async deleteProjectWithTasks(projectId: number): Promise<void> {
    // First get all related tasks
    const tasks = await this.service.getListItems<TaskItem>(
      "Tasks",
      ["Id"],
      undefined,
      `ProjectId eq ${projectId}`
    );

    // Delete all tasks
    await Promise.all(
      tasks.map((task) => this.service.deleteListItem("Tasks", task.Id))
    );

    // Delete the project
    await this.service.deleteListItem("Projects", projectId);
  }

  // Update related items
  async updateProjectStatus(
    projectId: number,
    status: ProjectItem["Status"]
  ): Promise<void> {
    // Update project status
    await this.service.updateListItem("Projects", projectId, {
      Status: status,
    });

    // If completed, mark all tasks as completed
    if (status === "Completed") {
      const tasks = await this.service.getListItems<TaskItem>(
        "Tasks",
        ["Id"],
        undefined,
        `ProjectId eq ${projectId} and IsCompleted eq false`
      );

      await Promise.all(
        tasks.map((task) =>
          this.service.updateListItem("Tasks", task.Id, { IsCompleted: true })
        )
      );
    }
  }
}
```

## Best Practices

### 1. Use Proper Field Selection

```typescript
// ✅ Good - Only select needed fields
const projects = await service.getListItems<ProjectItem>(
  "Projects",
  ["Id", "Title", "Status", "StartDate"],
  [],
  filter
);

// ❌ Avoid - Selecting all fields when not needed
const projects = await service.getListItems<ProjectItem>(
  "Projects",
  ["*"],
  [],
  filter
);
```

### 2. Handle Large Lists with Pagination

```typescript
// ✅ Good - Use pagination for large lists
const getAllProjectsPaginated = async (
  pageSize = 100
): Promise<ProjectItem[]> => {
  let allItems: ProjectItem[] = [];
  let hasNext = true;
  let skip = 0;

  while (hasNext) {
    const batch = await service.getListItems<ProjectItem>(
      "Projects",
      ["*"],
      [],
      undefined,
      "Id",
      pageSize
    );

    allItems = [...allItems, ...batch];
    hasNext = batch.length === pageSize;
    skip += pageSize;
  }

  return allItems;
};
```

### 3. Use Batch Operations for Multiple Updates

```typescript
// ✅ Good - Batch multiple operations
const updateMultipleProjects = async (
  updates: Array<{ id: number; data: Partial<ProjectItem> }>
) => {
  const batch = sp.web.createBatch();

  updates.forEach(({ id, data }) => {
    sp.web.lists
      .getByTitle("Projects")
      .items.getById(id)
      .inBatch(batch)
      .update(data);
  });

  await batch.execute();
};
```

### 4. Implement Proper Error Handling

```typescript
// ✅ Good - Comprehensive error handling
const safelyGetProject = async (id: number): Promise<ProjectItem | null> => {
  try {
    return await service.getListItemById<ProjectItem>("Projects", id);
  } catch (error) {
    if (error.status === 404) {
      return null; // Item not found
    }
    throw error; // Re-throw other errors
  }
};
```

## Next Steps

- [Data Fetching Patterns](./08-data-fetching-patterns.md)
- [SPFx API Usage](./09-spfx-api-usage.md)
- [Performance Optimization](./12-performance-optimization.md)
