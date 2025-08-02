# PostHog Analytics Integration for SharePoint Framework

This guide shows how to integrate PostHog analytics into your SharePoint Framework application for usage tracking, user behavior analysis, and feature adoption metrics.

## Important Considerations for SharePoint Environments

### 🔒 **Privacy & Compliance**

- **Enterprise Policies**: SharePoint is often used in enterprise environments with strict data policies
- **GDPR/Privacy Laws**: Ensure compliance with data protection regulations
- **User Consent**: Consider implementing consent mechanisms for analytics tracking
- **Data Residency**: PostHog offers EU cloud options for data sovereignty requirements

### 🏢 **SharePoint Context**

- **Multi-Tenant**: Your web part may be used across different SharePoint sites/tenants
- **User Identity**: SharePoint provides rich user context that can enhance analytics
- **Permission Awareness**: Track usage based on SharePoint permissions/roles
- **Environment Detection**: Different tracking for development vs production

## Implementation

### 1. Installation

```bash
npm install posthog-js
npm install -D @types/posthog-js
```

### 2. Create Analytics Configuration

```typescript
// src/config/analytics.ts
export const ANALYTICS_CONFIG = {
  // Enable/disable analytics entirely
  ENABLE_ANALYTICS: true, // Set to false to disable all tracking

  // PostHog configuration
  POSTHOG_KEY: process.env.POSTHOG_KEY || "your-posthog-key",
  POSTHOG_HOST: process.env.POSTHOG_HOST || "https://app.posthog.com",

  // Environment-based settings
  ENABLE_IN_DEVELOPMENT: false, // Usually keep false for dev
  ENABLE_IN_PRODUCTION: true,

  // Privacy settings
  RESPECT_DO_NOT_TRACK: true,
  ANONYMOUS_TRACKING: false, // Set true for anonymous tracking

  // SharePoint specific
  TRACK_SHAREPOINT_CONTEXT: true,
  TRACK_USER_PERMISSIONS: false, // Be careful with PII
} as const;

// Helper to determine if analytics should be active
export const shouldEnableAnalytics = (): boolean => {
  const isDev = process.env.NODE_ENV === "development";
  const isProd = process.env.NODE_ENV === "production";

  if (!ANALYTICS_CONFIG.ENABLE_ANALYTICS) return false;
  if (isDev && !ANALYTICS_CONFIG.ENABLE_IN_DEVELOPMENT) return false;
  if (isProd && !ANALYTICS_CONFIG.ENABLE_IN_PRODUCTION) return false;

  // Respect Do Not Track
  if (ANALYTICS_CONFIG.RESPECT_DO_NOT_TRACK && navigator.doNotTrack === "1") {
    return false;
  }

  return true;
};
```

### 3. Create PostHog Service

```typescript
// src/services/analytics.ts
import posthog from "posthog-js";
import { WebPartContext } from "@microsoft/sp-webpart-base";
import { ANALYTICS_CONFIG, shouldEnableAnalytics } from "../config/analytics";

export class AnalyticsService {
  private initialized = false;
  private context: WebPartContext | null = null;

  public initialize(context: WebPartContext): void {
    if (!shouldEnableAnalytics() || this.initialized) {
      return;
    }

    this.context = context;

    try {
      posthog.init(ANALYTICS_CONFIG.POSTHOG_KEY, {
        api_host: ANALYTICS_CONFIG.POSTHOG_HOST,

        // Privacy settings
        respect_dnt: ANALYTICS_CONFIG.RESPECT_DO_NOT_TRACK,
        disable_session_recording: true, // Usually disable for enterprise
        disable_persistence: ANALYTICS_CONFIG.ANONYMOUS_TRACKING,

        // SharePoint-friendly settings
        capture_pageview: false, // We'll handle this manually
        capture_pageleave: true,

        // Performance
        loaded: (posthog) => {
          if (ANALYTICS_CONFIG.ANONYMOUS_TRACKING) {
            posthog.opt_out_capturing(); // Start opted out, require explicit opt-in
          } else {
            this.identifyUser();
            this.setSharePointContext();
          }
        },
      });

      this.initialized = true;
      console.log("✅ Analytics initialized");
    } catch (error) {
      console.warn("⚠️ Analytics initialization failed:", error);
    }
  }

  private identifyUser(): void {
    if (!this.context || ANALYTICS_CONFIG.ANONYMOUS_TRACKING) return;

    try {
      const user = this.context.pageContext.user;

      // Use SharePoint user info for identification
      posthog.identify(user.loginName, {
        email: user.email,
        displayName: user.displayName,
        // Don't include sensitive info in production
        sharepoint_site: this.context.pageContext.web.title,
        sharepoint_tenant:
          this.context.pageContext.aadInfo?.tenantId?.toString(),
      });
    } catch (error) {
      console.warn("⚠️ User identification failed:", error);
    }
  }

  private setSharePointContext(): void {
    if (!this.context || !ANALYTICS_CONFIG.TRACK_SHAREPOINT_CONTEXT) return;

    try {
      posthog.register({
        // SharePoint environment context
        sp_site_url: this.context.pageContext.web.absoluteUrl,
        sp_site_title: this.context.pageContext.web.title,
        sp_web_template: this.context.pageContext.web.templateName,
        sp_page_type: this.context.pageContext.listItem ? "list_item" : "page",

        // Application context
        webpart_id: this.context.instanceId,
        webpart_version: this.context.manifest.version,
        spfx_version: this.context.sdks.microsoftTeams ? "teams" : "sharepoint",

        // User agent info (non-PII)
        is_mobile: /Mobile|Android|iPhone|iPad/.test(navigator.userAgent),
        browser_language: navigator.language,
      });
    } catch (error) {
      console.warn("⚠️ SharePoint context setting failed:", error);
    }
  }

  // Event tracking methods
  public trackEvent(eventName: string, properties?: Record<string, any>): void {
    if (!this.initialized || !shouldEnableAnalytics()) return;

    try {
      posthog.capture(eventName, {
        ...properties,
        timestamp: new Date().toISOString(),
        webpart_context: this.context?.instanceId,
      });
    } catch (error) {
      console.warn("⚠️ Event tracking failed:", error);
    }
  }

  public trackPageView(
    pageName: string,
    properties?: Record<string, any>
  ): void {
    this.trackEvent("page_view", {
      page_name: pageName,
      page_url: window.location.href,
      ...properties,
    });
  }

  public trackUserAction(
    action: string,
    properties?: Record<string, any>
  ): void {
    this.trackEvent("user_action", {
      action,
      ...properties,
    });
  }

  public trackFeatureUsage(
    feature: string,
    properties?: Record<string, any>
  ): void {
    this.trackEvent("feature_used", {
      feature_name: feature,
      ...properties,
    });
  }

  public trackError(error: Error, context?: Record<string, any>): void {
    this.trackEvent("error_occurred", {
      error_message: error.message,
      error_stack: error.stack,
      error_name: error.name,
      ...context,
    });
  }

  // SharePoint-specific tracking
  public trackListOperation(
    operation: "create" | "read" | "update" | "delete",
    listName: string,
    properties?: Record<string, any>
  ): void {
    this.trackEvent("sharepoint_list_operation", {
      operation,
      list_name: listName,
      ...properties,
    });
  }

  public trackSearchQuery(query: string, resultCount?: number): void {
    this.trackEvent("search_performed", {
      search_query: query.length > 50 ? query.substring(0, 50) + "..." : query, // Truncate long queries
      result_count: resultCount,
    });
  }

  // Consent management
  public optIn(): void {
    if (this.initialized) {
      posthog.opt_in_capturing();
      this.identifyUser();
      this.setSharePointContext();
    }
  }

  public optOut(): void {
    if (this.initialized) {
      posthog.opt_out_capturing();
    }
  }

  public isOptedOut(): boolean {
    return this.initialized ? posthog.has_opted_out_capturing() : true;
  }
}

// Singleton instance
export const analytics = new AnalyticsService();
```

### 4. Create Analytics React Hook

```typescript
// src/hooks/useAnalytics.ts
import { useCallback, useEffect } from "react";
import { analytics } from "../services/analytics";

export const useAnalytics = () => {
  const trackEvent = useCallback(
    (eventName: string, properties?: Record<string, any>) => {
      analytics.trackEvent(eventName, properties);
    },
    []
  );

  const trackPageView = useCallback(
    (pageName: string, properties?: Record<string, any>) => {
      analytics.trackPageView(pageName, properties);
    },
    []
  );

  const trackUserAction = useCallback(
    (action: string, properties?: Record<string, any>) => {
      analytics.trackUserAction(action, properties);
    },
    []
  );

  const trackFeatureUsage = useCallback(
    (feature: string, properties?: Record<string, any>) => {
      analytics.trackFeatureUsage(feature, properties);
    },
    []
  );

  const trackError = useCallback(
    (error: Error, context?: Record<string, any>) => {
      analytics.trackError(error, context);
    },
    []
  );

  const trackListOperation = useCallback(
    (
      operation: "create" | "read" | "update" | "delete",
      listName: string,
      properties?: Record<string, any>
    ) => {
      analytics.trackListOperation(operation, listName, properties);
    },
    []
  );

  return {
    trackEvent,
    trackPageView,
    trackUserAction,
    trackFeatureUsage,
    trackError,
    trackListOperation,

    // Consent methods
    optIn: analytics.optIn.bind(analytics),
    optOut: analytics.optOut.bind(analytics),
    isOptedOut: analytics.isOptedOut.bind(analytics),
  };
};

// Hook for automatic page view tracking
export const usePageTracking = (
  pageName: string,
  properties?: Record<string, any>
) => {
  const { trackPageView } = useAnalytics();

  useEffect(() => {
    trackPageView(pageName, properties);
  }, [trackPageView, pageName, properties]);
};
```

### 5. Initialize Analytics in Web Part

```typescript
// src/webparts/myApp/MyAppWebPart.ts
import * as React from "react";
import * as ReactDom from "react-dom";
import { BaseClientSideWebPart } from "@microsoft/sp-webpart-base";
import { AppRouter } from "./components/AppRouter";
import { analytics } from "../../services/analytics";
import { USE_SHAREPOINT_THEME } from "../../config/theme";

// Styles
import "../../styles/globals.css";
if (USE_SHAREPOINT_THEME) {
  import("../../styles/sharepoint-primitives.css");
}

export default class MyAppWebPart extends BaseClientSideWebPart<IMyAppWebPartProps> {
  public onInit(): Promise<void> {
    // Initialize analytics with SharePoint context
    analytics.initialize(this.context);

    return super.onInit();
  }

  public render(): void {
    const element: React.ReactElement = React.createElement(AppRouter, {
      context: this.context,
    });

    ReactDom.render(element, this.domElement);
  }

  protected onDispose(): void {
    ReactDom.unmountComponentAtNode(this.domElement);
  }
}
```

### 6. Usage in Components

```typescript
// src/components/ProjectsPage.tsx
import React from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { useAnalytics, usePageTracking } from "@/hooks/useAnalytics";
import { useSharePointList } from "@/hooks/useSharePointList";

export const ProjectsPage: React.FC<ProjectsPageProps> = ({ context }) => {
  const { trackUserAction, trackFeatureUsage, trackError } = useAnalytics();

  // Automatic page view tracking
  usePageTracking("projects_page", {
    page_section: "projects",
    user_role: context.pageContext.user.displayName
      ? "authenticated"
      : "anonymous",
  });

  const { items: projects, create: createProject } =
    useSharePointList<ProjectItem>({
      listName: "Projects",
      context,
    });

  const handleCreateProject = async (projectData: CreateProjectData) => {
    try {
      // Track the user action
      trackUserAction("create_project_clicked", {
        project_template: projectData.template,
      });

      const newProject = await createProject(projectData);

      // Track successful creation
      trackFeatureUsage("project_creation", {
        success: true,
        project_id: newProject.Id,
      });
    } catch (error) {
      // Track errors
      trackError(error as Error, {
        action: "create_project",
        project_data: projectData,
      });
    }
  };

  const handleProjectClick = (project: ProjectItem) => {
    trackUserAction("project_viewed", {
      project_id: project.Id,
      project_status: project.Status,
    });
  };

  return (
    <div>
      <div className="flex justify-between items-center mb-6">
        <h1>Projects</h1>
        <Button
          onClick={() => trackFeatureUsage("create_project_button_shown")}
        >
          Create Project
        </Button>
      </div>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {projects.map((project) => (
          <Card
            key={project.Id}
            className="cursor-pointer hover:shadow-md transition-shadow"
            onClick={() => handleProjectClick(project)}
          >
            <CardHeader>
              <CardTitle>{project.ProjectName}</CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-muted-foreground">{project.Description}</p>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  );
};
```

### 7. Error Boundary with Analytics

```typescript
// src/components/AnalyticsErrorBoundary.tsx
import React, { Component, ErrorInfo, ReactNode } from "react";
import { analytics } from "../services/analytics";

type AnalyticsErrorBoundaryProps = {
  children: ReactNode;
  fallback?: (error: Error, errorInfo: ErrorInfo) => ReactNode;
};

type AnalyticsErrorBoundaryState = {
  hasError: boolean;
  error?: Error;
};

export class AnalyticsErrorBoundary extends Component<
  AnalyticsErrorBoundaryProps,
  AnalyticsErrorBoundaryState
> {
  constructor(props: AnalyticsErrorBoundaryProps) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error): AnalyticsErrorBoundaryState {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // Track the error
    analytics.trackError(error, {
      error_boundary: true,
      component_stack: errorInfo.componentStack,
      error_info: errorInfo,
    });

    console.error("Error Boundary caught an error:", error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      if (this.props.fallback && this.state.error) {
        return this.props.fallback(this.state.error, {} as ErrorInfo);
      }

      return (
        <div className="p-4 bg-red-50 border border-red-200 rounded">
          <h2 className="text-red-800 font-semibold">Something went wrong</h2>
          <p className="text-red-600 mt-2">
            We've been notified of this error and are working to fix it.
          </p>
        </div>
      );
    }

    return this.props.children;
  }
}
```

## Privacy and Consent Management

### 1. Consent Banner Component

```typescript
// src/components/ConsentBanner.tsx
import React, { useState, useEffect } from "react";
import { Card, CardContent } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { useAnalytics } from "@/hooks/useAnalytics";

export const ConsentBanner: React.FC = () => {
  const [showBanner, setShowBanner] = useState(false);
  const { optIn, optOut, isOptedOut } = useAnalytics();

  useEffect(() => {
    // Check if user has already made a choice
    const hasConsent = localStorage.getItem("analytics-consent");
    if (!hasConsent && isOptedOut()) {
      setShowBanner(true);
    }
  }, [isOptedOut]);

  const handleAccept = () => {
    localStorage.setItem("analytics-consent", "accepted");
    optIn();
    setShowBanner(false);
  };

  const handleDecline = () => {
    localStorage.setItem("analytics-consent", "declined");
    optOut();
    setShowBanner(false);
  };

  if (!showBanner) return null;

  return (
    <div className="fixed bottom-4 right-4 max-w-md z-50">
      <Card>
        <CardContent className="p-4">
          <h3 className="font-semibold mb-2">Analytics & Cookies</h3>
          <p className="text-sm text-muted-foreground mb-4">
            We use analytics to improve your experience. Your data is processed
            securely and never shared with third parties.
          </p>
          <div className="flex gap-2">
            <Button size="sm" onClick={handleAccept}>
              Accept
            </Button>
            <Button variant="outline" size="sm" onClick={handleDecline}>
              Decline
            </Button>
          </div>
        </CardContent>
      </Card>
    </div>
  );
};
```

## Environment Configuration

### 1. Development vs Production

```typescript
// .env.development
POSTHOG_KEY=phc-dev-key-here
POSTHOG_HOST=https://app.posthog.com

// .env.production
POSTHOG_KEY=phc-prod-key-here
POSTHOG_HOST=https://eu.posthog.com  # EU instance if needed
```

### 2. Build-time Configuration

```javascript
// gulpfile.js - Add to your existing webpack config
const webpack = require("webpack");

build.configureWebpack.mergeConfig({
  additionalConfiguration: (generatedConfiguration) => {
    // Your existing PostCSS config...

    // Add environment variables
    generatedConfiguration.plugins.push(
      new webpack.DefinePlugin({
        "process.env.POSTHOG_KEY": JSON.stringify(process.env.POSTHOG_KEY),
        "process.env.POSTHOG_HOST": JSON.stringify(process.env.POSTHOG_HOST),
      })
    );

    return generatedConfiguration;
  },
});
```

## Useful Analytics Events to Track

### 1. User Engagement

- Page views and time spent
- Feature usage and adoption
- User flows through the application
- Search queries and results

### 2. SharePoint-Specific Events

- List operations (CRUD)
- Permission changes
- File uploads/downloads
- Integration usage (Teams, Graph, etc.)

### 3. Performance Metrics

- Load times
- Error rates
- User session duration
- Device/browser usage

### 4. Business Metrics

- Project creation/completion
- User collaboration patterns
- Feature adoption rates
- Support ticket correlation

## Best Practices

### ✅ **Privacy First**

- Always respect Do Not Track headers
- Implement proper consent management
- Anonymize or hash PII data
- Use EU PostHog instance if required

### ✅ **Enterprise Considerations**

- Work with IT/Security teams for approval
- Document what data is collected
- Provide opt-out mechanisms
- Consider on-premise PostHog for sensitive environments

### ✅ **Performance**

- Load PostHog asynchronously
- Don't block app functionality for analytics
- Batch events when possible
- Handle analytics failures gracefully

### ✅ **Data Quality**

- Consistent event naming conventions
- Include relevant context in all events
- Validate data before sending
- Monitor analytics implementation health

This setup gives you comprehensive usage analytics while respecting enterprise privacy requirements and SharePoint's security model!
