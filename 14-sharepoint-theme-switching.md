# SharePoint Theme Switching System

This guide shows how to create a flexible theme switching system that allows you to maintain your familiar shadcn/ui development experience while providing an option to switch to SharePoint native styling with a simple code flag.

## Overview

The system works by:

1. Using your existing shadcn/ui setup as the base
2. Creating a SharePoint primitives CSS file that overrides the same CSS variables
3. Using a simple boolean flag to toggle between themes
4. Requiring zero changes to your existing components

## Implementation

### 1. Create Theme Configuration

Create a simple configuration file for theme switching:

```typescript
// src/config/theme.ts
export const USE_SHAREPOINT_THEME = false; // Set to true for SharePoint native styling

// Optional: Export theme-aware utilities
export const getThemeClass = (
  sharepointClass: string,
  defaultClass: string
) => {
  return USE_SHAREPOINT_THEME ? sharepointClass : defaultClass;
};
```

### 2. Create SharePoint Primitives CSS File

Create a CSS file that overrides shadcn variables with SharePoint design tokens:

```css
/* src/styles/sharepoint-primitives.css */

:root {
  /* SharePoint Color Palette */
  /* Primary Colors - Based on SharePoint's themePrimary (#0078d4) */
  --primary: 213 94% 42%; /* #0078d4 - SharePoint primary blue */
  --primary-foreground: 0 0% 100%; /* White text on primary */

  /* Secondary Colors - Based on SharePoint's neutral palette */
  --secondary: 220 14% 96%; /* #f3f2f1 - SharePoint neutralLighter */
  --secondary-foreground: 32 13% 20%; /* #323130 - SharePoint neutralPrimary */

  /* Muted Colors - SharePoint neutral tones */
  --muted: 220 13% 91%; /* #edebe9 - SharePoint neutralLight */
  --muted-foreground: 220 9% 46%; /* #605e5c - SharePoint neutralSecondary */

  /* Accent Colors - SharePoint theme secondary */
  --accent: 217 91% 60%; /* #106ebe - SharePoint themeSecondary */
  --accent-foreground: 0 0% 100%; /* White text on accent */

  /* Background Colors */
  --background: 0 0% 100%; /* Pure white - SharePoint page background */
  --foreground: 32 13% 20%; /* #323130 - SharePoint neutralPrimary */

  /* Border and Input Colors */
  --border: 220 13% 64%; /* #a19f9d - SharePoint neutralTertiary */
  --input: 220 13% 91%; /* #edebe9 - SharePoint neutralLight */
  --ring: 213 94% 42%; /* #0078d4 - SharePoint primary for focus rings */

  /* Card Colors */
  --card: 0 0% 100%; /* White cards */
  --card-foreground: 32 13% 20%; /* Dark text on cards */

  /* Popover Colors */
  --popover: 0 0% 100%; /* White popover background */
  --popover-foreground: 32 13% 20%; /* Dark text in popovers */

  /* Destructive Colors - SharePoint error red */
  --destructive: 0 65% 51%; /* #d13438 - SharePoint error color */
  --destructive-foreground: 0 0% 100%; /* White text on error */

  /* SharePoint Specific Colors (for custom usage) */
  --sp-theme-primary: 213 94% 42%; /* #0078d4 */
  --sp-theme-secondary: 217 91% 60%; /* #106ebe */
  --sp-theme-tertiary: 221 87% 80%; /* #005a9e */
  --sp-theme-light: 213 76% 87%; /* #c7e0f4 */
  --sp-theme-lighter: 213 56% 94%; /* #deecf9 */
  --sp-theme-lighter-alt: 213 36% 98%; /* #eff6fc */

  --sp-neutral-primary: 32 13% 20%; /* #323130 */
  --sp-neutral-secondary: 30 7% 38%; /* #605e5c */
  --sp-neutral-tertiary: 25 6% 64%; /* #a19f9d */
  --sp-neutral-quaternary: 22 5% 78%; /* #c8c6c4 */
  --sp-neutral-tertiary-alt: 24 6% 82%; /* #d2d0ce */
  --sp-neutral-light: 30 4% 93%; /* #edebe9 */
  --sp-neutral-lighter: 30 3% 95%; /* #f3f2f1 */
  --sp-neutral-lighter-alt: 30 2% 97%; /* #faf9f8 */

  /* SharePoint Typography */
  --font-sans: "Segoe UI", "Segoe UI Web (West European)", "Segoe UI",
    -apple-system, BlinkMacSystemFont, Roboto, "Helvetica Neue", sans-serif;

  /* SharePoint Spacing Scale */
  --sp-spacing-xs: 4px; /* Extra small spacing */
  --sp-spacing-s: 8px; /* Small spacing */
  --sp-spacing-m: 16px; /* Medium spacing */
  --sp-spacing-l: 20px; /* Large spacing */
  --sp-spacing-xl: 32px; /* Extra large spacing */

  /* SharePoint Border Radius */
  --radius: 0.125rem; /* 2px - SharePoint uses smaller radius */
}

/* SharePoint-specific component overrides */
@layer components {
  /* Ensure SharePoint font is applied */
  * {
    font-family: var(--font-sans);
  }

  /* SharePoint button styling adjustments */
  .btn {
    font-weight: 400; /* SharePoint uses normal weight, not medium */
    letter-spacing: normal; /* Remove any letter spacing */
  }

  /* SharePoint card styling */
  .card {
    box-shadow: 0 1.6px 3.6px 0 rgba(0, 0, 0, 0.132), 0 0.3px 0.9px 0 rgba(0, 0, 0, 0.108); /* SharePoint card shadow */
  }

  /* SharePoint input styling */
  .input {
    border: 1px solid var(--border);
    font-weight: 400;
  }

  .input:focus {
    border-color: var(--ring);
    box-shadow: inset 0 0 0 1px var(--ring);
  }
}

/* Dark theme support (SharePoint dark theme) */
[data-theme="dark"] {
  --background: 24 9% 10%; /* #1b1a19 - SharePoint dark background */
  --foreground: 0 0% 100%; /* White text in dark mode */
  --card: 24 9% 10%; /* Dark card background */
  --card-foreground: 0 0% 100%; /* White text on cards */
  --popover: 24 9% 10%; /* Dark popover background */
  --popover-foreground: 0 0% 100%; /* White text in popovers */
  --primary: 213 94% 42%; /* Keep SharePoint blue in dark mode */
  --primary-foreground: 0 0% 100%; /* White text on primary */
  --secondary: 24 6% 20%; /* Dark secondary background */
  --secondary-foreground: 0 0% 100%; /* White text on secondary */
  --muted: 24 6% 20%; /* Dark muted background */
  --muted-foreground: 220 5% 65%; /* Light gray text */
  --accent: 24 6% 20%; /* Dark accent background */
  --accent-foreground: 0 0% 100%; /* White text on accent */
  --destructive: 0 65% 51%; /* Keep error red in dark mode */
  --destructive-foreground: 0 0% 100%; /* White text on error */
  --border: 24 6% 20%; /* Dark borders */
  --input: 24 6% 20%; /* Dark input background */
  --ring: 213 94% 42%; /* Keep SharePoint blue for focus */
}
```

### 3. Update Tailwind Configuration

Add SharePoint spacing and colors to your existing Tailwind config:

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      // Your existing shadcn colors...
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },

        // Add SharePoint-specific colors
        sharepoint: {
          primary: "hsl(var(--sp-theme-primary))",
          secondary: "hsl(var(--sp-theme-secondary))",
          tertiary: "hsl(var(--sp-theme-tertiary))",
          light: "hsl(var(--sp-theme-light))",
          lighter: "hsl(var(--sp-theme-lighter))",
          lighterAlt: "hsl(var(--sp-theme-lighter-alt))",
        },
        neutral: {
          primary: "hsl(var(--sp-neutral-primary))",
          secondary: "hsl(var(--sp-neutral-secondary))",
          tertiary: "hsl(var(--sp-neutral-tertiary))",
          quaternary: "hsl(var(--sp-neutral-quaternary))",
          tertiaryAlt: "hsl(var(--sp-neutral-tertiary-alt))",
          light: "hsl(var(--sp-neutral-light))",
          lighter: "hsl(var(--sp-neutral-lighter))",
          lighterAlt: "hsl(var(--sp-neutral-lighter-alt))",
        },
      },

      // Add SharePoint spacing
      spacing: {
        // Your existing spacing...
        "sp-xs": "var(--sp-spacing-xs)",
        "sp-s": "var(--sp-spacing-s)",
        "sp-m": "var(--sp-spacing-m)",
        "sp-l": "var(--sp-spacing-l)",
        "sp-xl": "var(--sp-spacing-xl)",
      },

      // SharePoint font family
      fontFamily: {
        sans: ["var(--font-sans)", "system-ui", "sans-serif"],
        segoe: ["var(--font-sans)"],
      },

      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
    },
  },
  plugins: [require("@tailwindcss/typography"), require("@tailwindcss/forms")],
};
```

### 4. Implement Theme Switching in Your Web Part

Update your main web part file to conditionally load the SharePoint theme:

```typescript
// src/webparts/myApp/MyAppWebPart.ts
import * as React from "react";
import * as ReactDom from "react-dom";
import { BaseClientSideWebPart } from "@microsoft/sp-webpart-base";
import { AppRouter } from "./components/AppRouter";
import { USE_SHAREPOINT_THEME } from "../../config/theme";

// Always import the base styles
import "../../styles/globals.css";

// Conditionally import SharePoint primitives
if (USE_SHAREPOINT_THEME) {
  import("../../styles/sharepoint-primitives.css");
}

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
}
```

### 5. Optional: Create Theme-Aware Utility Functions

For advanced usage, you can create utilities that are theme-aware:

```typescript
// src/lib/theme-utils.ts
import { USE_SHAREPOINT_THEME } from "../config/theme";
import { cn } from "./utils";

/**
 * Apply SharePoint-specific classes when SharePoint theme is active
 */
export const spClass = (
  sharepointClass: string,
  defaultClass: string = ""
): string => {
  return USE_SHAREPOINT_THEME ? sharepointClass : defaultClass;
};

/**
 * Conditional class names based on theme
 */
export const themeClass = (classes: {
  sharepoint?: string;
  default?: string;
  both?: string;
}): string => {
  return cn(
    classes.both,
    USE_SHAREPOINT_THEME ? classes.sharepoint : classes.default
  );
};

/**
 * Get appropriate spacing class for current theme
 */
export const spSpacing = {
  xs: USE_SHAREPOINT_THEME ? "sp-xs" : "p-1",
  s: USE_SHAREPOINT_THEME ? "sp-s" : "p-2",
  m: USE_SHAREPOINT_THEME ? "sp-m" : "p-4",
  l: USE_SHAREPOINT_THEME ? "sp-l" : "p-5",
  xl: USE_SHAREPOINT_THEME ? "sp-xl" : "p-8",
};
```

## Usage Examples

### Basic Component (No Changes Needed)

Your existing shadcn components work with both themes without any modifications:

```typescript
// src/components/ProjectCard.tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";

// This component automatically adapts to the current theme
export const ProjectCard: React.FC<ProjectCardProps> = ({ project }) => {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{project.title}</CardTitle>
        <Badge variant="secondary">{project.status}</Badge>
      </CardHeader>
      <CardContent>
        <p className="text-muted-foreground">{project.description}</p>
        <div className="flex gap-2 mt-4">
          <Button variant="default">Edit</Button>
          <Button variant="outline">View</Button>
        </div>
      </CardContent>
    </Card>
  );
};
```

### Advanced Component with Theme-Specific Styling

For components that need theme-specific behavior:

```typescript
// src/components/SharePointOptimizedCard.tsx
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { themeClass, spSpacing } from "@/lib/theme-utils";

export const SharePointOptimizedCard: React.FC<ProjectCardProps> = ({
  project,
}) => {
  return (
    <Card
      className={themeClass({
        sharepoint: "font-segoe shadow-sm", // SharePoint-specific styling
        default: "shadow-md", // Regular shadcn styling
        both: "border-border", // Applied to both themes
      })}
    >
      <CardHeader className={`space-y-${spSpacing.s}`}>
        <CardTitle
          className={themeClass({
            sharepoint: "text-neutral-primary font-normal",
            default: "font-semibold",
          })}
        >
          {project.title}
        </CardTitle>
      </CardHeader>
      <CardContent>
        <Button
          variant="default"
          className={themeClass({
            sharepoint: "bg-sharepoint-primary hover:bg-sharepoint-secondary",
            default: "",
          })}
        >
          Edit Project
        </Button>
      </CardContent>
    </Card>
  );
};
```

## Development Workflow

### 1. Development Phase

```typescript
// src/config/theme.ts
export const USE_SHAREPOINT_THEME = false; // Use familiar shadcn styling
```

- Develop with your familiar shadcn/ui components
- All components work normally
- Fast iteration and development

### 2. SharePoint Testing Phase

```typescript
// src/config/theme.ts
export const USE_SHAREPOINT_THEME = true; // Switch to SharePoint styling
```

- Test how your app looks with SharePoint native styling
- Verify theme integration works correctly
- Make any theme-specific adjustments

### 3. Production Deployment

```typescript
// src/config/theme.ts
export const USE_SHAREPOINT_THEME = true; // Deploy with SharePoint styling
```

- Deploy to SharePoint with native styling
- Users get familiar SharePoint look and feel
- Maintains all your component functionality

## Benefits

### ✅ **Developer Experience**

- Keep your familiar shadcn/ui development workflow
- No component changes required
- Simple one-line toggle

### ✅ **User Experience**

- Native SharePoint styling when deployed
- Consistent with SharePoint design language
- Automatic dark mode support

### ✅ **Maintenance**

- Single codebase for both themes
- CSS-only switching (no JavaScript overhead)
- Easy to update SharePoint colors when needed

### ✅ **Performance**

- No runtime theme switching logic
- Build-time CSS optimization
- Smaller bundle size (only one theme loaded)

## Best Practices

1. **Start Development with `USE_SHAREPOINT_THEME = false`** for familiar styling during development

2. **Test Regularly with SharePoint Theme** by toggling the flag to ensure compatibility

3. **Use Theme-Agnostic Component APIs** - avoid hardcoding colors or spacing in component props

4. **Leverage CSS Variables** - both themes use the same variable names, ensuring compatibility

5. **Deploy with SharePoint Theme** for production to meet user expectations

This system gives you the best of both worlds: productive development with familiar tools and native SharePoint styling for end users.
