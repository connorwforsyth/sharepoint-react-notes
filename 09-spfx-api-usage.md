# SharePoint Framework API Usage

This guide covers advanced SharePoint Framework API patterns, including Graph API integration, PnPjs usage, and context management.

## WebPart Context Management

### Context Provider Pattern

```typescript
// src/contexts/SPFxContext.tsx
import React, { createContext, useContext, ReactNode } from "react";
import { WebPartContext } from "@microsoft/sp-webpart-base";

type SPFxContextType = {
  context: WebPartContext;
  siteUrl: string;
  webUrl: string;
  userDisplayName: string;
  userEmail: string;
  isTeamsContext: boolean;
};

const SPFxContext = createContext<SPFxContextType | undefined>(undefined);

type SPFxProviderProps = {
  context: WebPartContext;
  children: ReactNode;
};

export const SPFxProvider: React.FC<SPFxProviderProps> = ({
  context,
  children,
}) => {
  const contextValue: SPFxContextType = {
    context,
    siteUrl: context.pageContext.site.absoluteUrl,
    webUrl: context.pageContext.web.absoluteUrl,
    userDisplayName: context.pageContext.user.displayName,
    userEmail: context.pageContext.user.email,
    isTeamsContext: !!context.sdks?.microsoftTeams,
  };

  return (
    <SPFxContext.Provider value={contextValue}>{children}</SPFxContext.Provider>
  );
};

export const useSPFxContext = (): SPFxContextType => {
  const context = useContext(SPFxContext);
  if (!context) {
    throw new Error("useSPFxContext must be used within SPFxProvider");
  }
  return context;
};

// Specialized hooks
export const useCurrentUser = () => {
  const { context, userDisplayName, userEmail } = useSPFxContext();

  return {
    displayName: userDisplayName,
    email: userEmail,
    loginName: context.pageContext.user.loginName,
    id: context.pageContext.user.loginName,
  };
};

export const useSiteInfo = () => {
  const { context, siteUrl, webUrl } = useSPFxContext();

  return {
    siteUrl,
    webUrl,
    siteId: context.pageContext.site.id.toString(),
    webId: context.pageContext.web.id.toString(),
    siteName: context.pageContext.web.title,
  };
};
```

## Microsoft Graph Integration

### Graph Client Setup

```typescript
// src/lib/graphClient.ts
import { MSGraphClientV3 } from "@microsoft/sp-http";
import { WebPartContext } from "@microsoft/sp-webpart-base";

export class GraphService {
  private graphClient: MSGraphClientV3;

  constructor(context: WebPartContext) {
    this.graphClient = context.serviceScope.consume(MSGraphClientV3.serviceKey);
  }

  // User operations
  async getCurrentUser() {
    try {
      const user = await this.graphClient
        .api("/me")
        .select("id,displayName,mail,userPrincipalName,jobTitle,department")
        .get();
      return user;
    } catch (error) {
      console.error("Error fetching current user:", error);
      throw error;
    }
  }

  async getUserPhoto(userId?: string): Promise<string | null> {
    try {
      const endpoint = userId
        ? `/users/${userId}/photo/$value`
        : "/me/photo/$value";
      const photoBlob = await this.graphClient
        .api(endpoint)
        .responseType("blob")
        .get();

      return URL.createObjectURL(photoBlob);
    } catch (error) {
      console.warn("Error fetching user photo:", error);
      return null;
    }
  }

  // Teams operations
  async getMyTeams() {
    try {
      const response = await this.graphClient
        .api("/me/joinedTeams")
        .select("id,displayName,description,webUrl")
        .get();
      return response.value;
    } catch (error) {
      console.error("Error fetching teams:", error);
      throw error;
    }
  }

  async getTeamChannels(teamId: string) {
    try {
      const response = await this.graphClient
        .api(`/teams/${teamId}/channels`)
        .select("id,displayName,description,webUrl")
        .get();
      return response.value;
    } catch (error) {
      console.error("Error fetching team channels:", error);
      throw error;
    }
  }

  // Calendar operations
  async getCalendarEvents(startDate: Date, endDate: Date) {
    try {
      const response = await this.graphClient
        .api("/me/events")
        .filter(
          `start/dateTime ge '${startDate.toISOString()}' and end/dateTime le '${endDate.toISOString()}'`
        )
        .select("id,subject,start,end,location,attendees")
        .orderby("start/dateTime")
        .top(50)
        .get();
      return response.value;
    } catch (error) {
      console.error("Error fetching calendar events:", error);
      throw error;
    }
  }

  // Files operations
  async getRecentFiles() {
    try {
      const response = await this.graphClient
        .api("/me/drive/recent")
        .select("id,name,lastModifiedDateTime,webUrl,createdBy")
        .top(20)
        .get();
      return response.value;
    } catch (error) {
      console.error("Error fetching recent files:", error);
      throw error;
    }
  }

  async searchFiles(query: string) {
    try {
      const response = await this.graphClient
        .api("/me/drive/root/search")
        .query({ q: query })
        .select("id,name,lastModifiedDateTime,webUrl,folder,file")
        .top(25)
        .get();
      return response.value;
    } catch (error) {
      console.error("Error searching files:", error);
      throw error;
    }
  }

  // Batch operations
  async batchRequest(requests: any[]) {
    try {
      const batch = {
        requests: requests.map((req, index) => ({
          id: index.toString(),
          ...req,
        })),
      };

      const response = await this.graphClient.api("/$batch").post(batch);

      return response.responses;
    } catch (error) {
      console.error("Error executing batch request:", error);
      throw error;
    }
  }
}
```

### Graph Hooks

```typescript
// src/hooks/useGraph.ts
import { useState, useEffect, useCallback } from "react";
import { GraphService } from "@/lib/graphClient";
import { useSPFxContext } from "@/contexts/SPFxContext";

export const useCurrentUser = () => {
  const { context } = useSPFxContext();
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const loadUser = async () => {
      try {
        const graphService = new GraphService(context);
        const userData = await graphService.getCurrentUser();
        setUser(userData);
      } catch (err) {
        setError(err instanceof Error ? err.message : "Failed to load user");
      } finally {
        setLoading(false);
      }
    };

    loadUser();
  }, [context]);

  return { user, loading, error };
};

export const useUserPhoto = (userId?: string) => {
  const { context } = useSPFxContext();
  const [photoUrl, setPhotoUrl] = useState<string | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const loadPhoto = async () => {
      try {
        const graphService = new GraphService(context);
        const url = await graphService.getUserPhoto(userId);
        setPhotoUrl(url);
      } catch (error) {
        console.warn("Failed to load user photo:", error);
      } finally {
        setLoading(false);
      }
    };

    loadPhoto();
  }, [context, userId]);

  // Cleanup object URL on unmount
  useEffect(() => {
    return () => {
      if (photoUrl) {
        URL.revokeObjectURL(photoUrl);
      }
    };
  }, [photoUrl]);

  return { photoUrl, loading };
};

export const useCalendarEvents = (startDate: Date, endDate: Date) => {
  const { context } = useSPFxContext();
  const [events, setEvents] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const loadEvents = useCallback(async () => {
    try {
      setLoading(true);
      setError(null);
      const graphService = new GraphService(context);
      const eventData = await graphService.getCalendarEvents(
        startDate,
        endDate
      );
      setEvents(eventData);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Failed to load events");
    } finally {
      setLoading(false);
    }
  }, [context, startDate, endDate]);

  useEffect(() => {
    loadEvents();
  }, [loadEvents]);

  return { events, loading, error, refresh: loadEvents };
};
```

## PnPjs Integration

### PnPjs Service Layer

```typescript
// src/lib/pnpService.ts
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { spfi, SPFx } from "@pnp/sp";
import { graphfi, GraphFI, SPFx as GraphSPFx } from "@pnp/graph";
import "@pnp/sp/webs";
import "@pnp/sp/lists";
import "@pnp/sp/items";
import "@pnp/sp/fields";
import "@pnp/sp/site-users/web";
import "@pnp/sp/profiles";

export class PnPService {
  private sp: ReturnType<typeof spfi>;
  private graph: GraphFI;

  constructor(context: WebPartContext) {
    this.sp = spfi().using(SPFx(context));
    this.graph = graphfi().using(GraphSPFx(context));
  }

  // Web operations
  async getWebProperties() {
    try {
      return await this.sp.web.select("*", "RegionalSettings/TimeZone")();
    } catch (error) {
      console.error("Error fetching web properties:", error);
      throw error;
    }
  }

  async getCurrentUserGroups() {
    try {
      const groups = await this.sp.web.currentUser.groups();
      return groups;
    } catch (error) {
      console.error("Error fetching user groups:", error);
      throw error;
    }
  }

  // Advanced list operations
  async getListWithFields(listTitle: string) {
    try {
      const [list, fields] = await Promise.all([
        this.sp.web.lists.getByTitle(listTitle)(),
        this.sp.web.lists
          .getByTitle(listTitle)
          .fields.filter("Hidden eq false")(),
      ]);

      return { list, fields };
    } catch (error) {
      console.error(`Error fetching list ${listTitle}:`, error);
      throw error;
    }
  }

  async getItemsWithPaging(
    listTitle: string,
    pageSize: number = 100,
    select?: string[],
    filter?: string
  ) {
    try {
      let query = this.sp.web.lists.getByTitle(listTitle).items.top(pageSize);

      if (select) {
        query = query.select(...select);
      }

      if (filter) {
        query = query.filter(filter);
      }

      const result = await query.getPaged();

      return {
        items: result.results,
        hasNext: result.hasNext,
        getNext: result.getNext,
      };
    } catch (error) {
      console.error("Error fetching paged items:", error);
      throw error;
    }
  }

  // Search operations
  async searchContent(query: string, sourceId?: string) {
    try {
      let searchQuery = this.sp.search(query);

      if (sourceId) {
        searchQuery = searchQuery.sourceId(sourceId);
      }

      const results = await searchQuery();
      return results;
    } catch (error) {
      console.error("Error searching content:", error);
      throw error;
    }
  }

  // Taxonomy operations
  async getTermSets() {
    try {
      const termStore = this.sp.termStore;
      const sets = await termStore.sets();
      return sets;
    } catch (error) {
      console.error("Error fetching term sets:", error);
      throw error;
    }
  }

  async getTermsByTermSet(termSetId: string) {
    try {
      const terms = await this.sp.termStore.sets.getById(termSetId).terms();
      return terms;
    } catch (error) {
      console.error("Error fetching terms:", error);
      throw error;
    }
  }

  // User Profile operations
  async getUserProfile(loginName?: string) {
    try {
      const profiles = this.sp.profiles;
      const profile = loginName
        ? await profiles.getPropertiesFor(loginName)
        : await profiles.myProperties();

      return profile;
    } catch (error) {
      console.error("Error fetching user profile:", error);
      throw error;
    }
  }

  // Batch operations with PnP
  async executeBatch(operations: Array<() => Promise<any>>) {
    try {
      const batch = this.sp.web.batched();
      const promises = operations.map((op) => op());

      await batch.execute();
      const results = await Promise.all(promises);
      return results;
    } catch (error) {
      console.error("Error executing batch operations:", error);
      throw error;
    }
  }
}
```

## HTTP Client Usage

### Custom HTTP Service

```typescript
// src/lib/httpService.ts
import {
  HttpClient,
  IHttpClientOptions,
  HttpClientResponse,
} from "@microsoft/sp-http";
import { WebPartContext } from "@microsoft/sp-webpart-base";

export class SPHttpService {
  private httpClient: HttpClient;
  private context: WebPartContext;

  constructor(context: WebPartContext) {
    this.context = context;
    this.httpClient = context.httpClient;
  }

  private getBaseHeaders(): Record<string, string> {
    return {
      Accept: "application/json;odata=verbose",
      "Content-Type": "application/json;odata=verbose",
      "X-RequestDigest":
        this.context.pageContext.legacyPageContext.formDigestValue,
    };
  }

  async get<T>(url: string, options?: IHttpClientOptions): Promise<T> {
    try {
      const response: HttpClientResponse = await this.httpClient.get(
        url,
        HttpClient.configurations.v1,
        {
          ...options,
          headers: {
            ...this.getBaseHeaders(),
            ...options?.headers,
          },
        }
      );

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const data = await response.json();
      return data.d ? data.d : data;
    } catch (error) {
      console.error(`GET request failed for ${url}:`, error);
      throw error;
    }
  }

  async post<T>(
    url: string,
    body: any,
    options?: IHttpClientOptions
  ): Promise<T> {
    try {
      const response: HttpClientResponse = await this.httpClient.post(
        url,
        HttpClient.configurations.v1,
        {
          ...options,
          body: JSON.stringify(body),
          headers: {
            ...this.getBaseHeaders(),
            ...options?.headers,
          },
        }
      );

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const data = await response.json();
      return data.d ? data.d : data;
    } catch (error) {
      console.error(`POST request failed for ${url}:`, error);
      throw error;
    }
  }

  async patch<T>(
    url: string,
    body: any,
    options?: IHttpClientOptions
  ): Promise<T> {
    try {
      const response: HttpClientResponse = await this.httpClient.fetch(
        url,
        HttpClient.configurations.v1,
        {
          ...options,
          method: "PATCH",
          body: JSON.stringify(body),
          headers: {
            ...this.getBaseHeaders(),
            "IF-MATCH": "*",
            ...options?.headers,
          },
        }
      );

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      // PATCH requests often return empty response
      if (response.status === 204) {
        return {} as T;
      }

      const data = await response.json();
      return data.d ? data.d : data;
    } catch (error) {
      console.error(`PATCH request failed for ${url}:`, error);
      throw error;
    }
  }

  async delete(url: string, options?: IHttpClientOptions): Promise<void> {
    try {
      const response: HttpClientResponse = await this.httpClient.fetch(
        url,
        HttpClient.configurations.v1,
        {
          ...options,
          method: "DELETE",
          headers: {
            ...this.getBaseHeaders(),
            "IF-MATCH": "*",
            ...options?.headers,
          },
        }
      );

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
    } catch (error) {
      console.error(`DELETE request failed for ${url}:`, error);
      throw error;
    }
  }

  // SharePoint specific operations
  async getFormDigest(): Promise<string> {
    try {
      const response = await this.post(
        `${this.context.pageContext.web.absoluteUrl}/_api/contextinfo`,
        {}
      );
      return response.FormDigestValue;
    } catch (error) {
      console.error("Failed to get form digest:", error);
      throw error;
    }
  }

  async executeSearch(query: string): Promise<any> {
    try {
      const searchUrl = `${
        this.context.pageContext.web.absoluteUrl
      }/_api/search/query?querytext='${encodeURIComponent(query)}'`;
      const response = await this.get(searchUrl);
      return response.PrimaryQueryResult?.RelevantResults?.Table?.Rows || [];
    } catch (error) {
      console.error("Search request failed:", error);
      throw error;
    }
  }
}
```

## Teams Integration

### Teams Context Hook

```typescript
// src/hooks/useTeamsContext.ts
import { useState, useEffect } from "react";
import { useSPFxContext } from "@/contexts/SPFxContext";

type TeamsContextInfo = {
  isInTeams: boolean;
  teamId?: string;
  channelId?: string;
  chatId?: string;
  teamName?: string;
  channelName?: string;
  theme?: string;
};

export const useTeamsContext = (): TeamsContextInfo => {
  const { context } = useSPFxContext();
  const [teamsInfo, setTeamsInfo] = useState<TeamsContextInfo>({
    isInTeams: false,
  });

  useEffect(() => {
    const loadTeamsContext = async () => {
      if (context.sdks?.microsoftTeams) {
        try {
          const teamsContext =
            await context.sdks.microsoftTeams.teamsJs.app.getContext();

          setTeamsInfo({
            isInTeams: true,
            teamId: teamsContext.team?.groupId,
            channelId: teamsContext.channel?.id,
            chatId: teamsContext.chat?.id,
            teamName: teamsContext.team?.displayName,
            channelName: teamsContext.channel?.displayName,
            theme: teamsContext.app.theme,
          });
        } catch (error) {
          console.error("Error loading Teams context:", error);
        }
      }
    };

    loadTeamsContext();
  }, [context]);

  return teamsInfo;
};

// Teams theme hook
export const useTeamsTheme = () => {
  const { context } = useSPFxContext();
  const [theme, setTheme] = useState<string>("default");

  useEffect(() => {
    if (context.sdks?.microsoftTeams) {
      const handleThemeChange = (newTheme: string) => {
        setTheme(newTheme);

        // Apply theme to CSS variables
        const root = document.documentElement;
        switch (newTheme) {
          case "dark":
            root.classList.add("teams-dark");
            root.classList.remove("teams-contrast");
            break;
          case "contrast":
            root.classList.add("teams-contrast");
            root.classList.remove("teams-dark");
            break;
          default:
            root.classList.remove("teams-dark", "teams-contrast");
        }
      };

      // Set initial theme
      context.sdks.microsoftTeams.teamsJs.app
        .getContext()
        .then((teamsContext) => {
          handleThemeChange(teamsContext.app.theme);
        });

      // Listen for theme changes
      context.sdks.microsoftTeams.teamsJs.app.registerOnThemeChangeHandler(
        handleThemeChange
      );

      return () => {
        // Cleanup if needed
      };
    }
  }, [context]);

  return theme;
};
```

## Permission Management

### Permission Service

```typescript
// src/lib/permissionService.ts
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { PnPService } from "./pnpService";

export enum SPPermission {
  ViewListItems = "ViewListItems",
  AddListItems = "AddListItems",
  EditListItems = "EditListItems",
  DeleteListItems = "DeleteListItems",
  ManageLists = "ManageLists",
  FullControl = "FullControl",
}

export class PermissionService {
  private pnpService: PnPService;

  constructor(context: WebPartContext) {
    this.pnpService = new PnPService(context);
  }

  async hasPermission(
    permission: SPPermission,
    listTitle?: string
  ): Promise<boolean> {
    try {
      if (listTitle) {
        const effectivePerms = await this.pnpService.sp.web.lists
          .getByTitle(listTitle)
          .getCurrentUserEffectivePermissions();

        return this.pnpService.sp.web.lists
          .getByTitle(listTitle)
          .userHasPermissions(effectivePerms, permission);
      } else {
        const effectivePerms =
          await this.pnpService.sp.web.getCurrentUserEffectivePermissions();

        return this.pnpService.sp.web.userHasPermissions(
          effectivePerms,
          permission
        );
      }
    } catch (error) {
      console.error("Error checking permissions:", error);
      return false;
    }
  }

  async getUserPermissions(listTitle?: string): Promise<string[]> {
    try {
      if (listTitle) {
        const perms = await this.pnpService.sp.web.lists
          .getByTitle(listTitle)
          .getCurrentUserEffectivePermissions();
        return perms.value;
      } else {
        const perms =
          await this.pnpService.sp.web.getCurrentUserEffectivePermissions();
        return perms.value;
      }
    } catch (error) {
      console.error("Error fetching user permissions:", error);
      return [];
    }
  }

  async isUserInGroup(groupName: string): Promise<boolean> {
    try {
      const groups = await this.pnpService.sp.web.currentUser.groups();
      return groups.some((group) => group.Title === groupName);
    } catch (error) {
      console.error("Error checking group membership:", error);
      return false;
    }
  }
}

// Permission hook
export const usePermissions = (listTitle?: string) => {
  const { context } = useSPFxContext();
  const [permissions, setPermissions] = useState<string[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const loadPermissions = async () => {
      try {
        const permissionService = new PermissionService(context);
        const userPermissions = await permissionService.getUserPermissions(
          listTitle
        );
        setPermissions(userPermissions);
      } catch (error) {
        console.error("Failed to load permissions:", error);
      } finally {
        setLoading(false);
      }
    };

    loadPermissions();
  }, [context, listTitle]);

  const hasPermission = useCallback(
    (permission: SPPermission) => {
      return permissions.includes(permission);
    },
    [permissions]
  );

  return { permissions, hasPermission, loading };
};
```

## Best Practices

### 1. Use Service Scoped Dependencies

```typescript
// ✅ Good - Use service scope
export class MyService {
  constructor(
    private context: WebPartContext,
    private httpClient: HttpClient = context.httpClient
  ) {}
}
```

### 2. Handle Context Changes

```typescript
// ✅ Good - Handle context updates
useEffect(() => {
  const handleContextChange = () => {
    // Reinitialize services with new context
    initializeServices();
  };

  context.serviceScope.whenFinished(() => {
    handleContextChange();
  });
}, [context]);
```

### 3. Implement Proper Error Handling

```typescript
// ✅ Good - Comprehensive error handling
const safeApiCall = async <T>(
  apiCall: () => Promise<T>,
  fallback: T
): Promise<T> => {
  try {
    return await apiCall();
  } catch (error) {
    if (error.status === 403) {
      console.warn("Access denied - using fallback");
      return fallback;
    }
    throw error;
  }
};
```

### 4. Cache Service Instances

```typescript
// ✅ Good - Singleton pattern for services
class ServiceManager {
  private static instances = new Map<string, any>();

  static getService<T>(key: string, factory: () => T): T {
    if (!this.instances.has(key)) {
      this.instances.set(key, factory());
    }
    return this.instances.get(key);
  }
}
```

## Next Steps

- [Development Workflow](./10-development-workflow.md)
- [Testing Strategies](./11-testing-strategies.md)
- [Performance Optimization](./12-performance-optimization.md)
