# Tailwind CSS & shadcn/ui Setup

This guide covers setting up Tailwind CSS and shadcn/ui components in your SharePoint Framework project for modern, accessible styling.

## Tailwind CSS Configuration

### 1. Install Dependencies

```bash
# Core Tailwind CSS
npm install -D tailwindcss postcss autoprefixer

# Additional Tailwind plugins
npm install -D @tailwindcss/typography @tailwindcss/forms @tailwindcss/aspect-ratio

# Utility libraries
npm install clsx tailwind-merge class-variance-authority

# Icon library
npm install lucide-react
```

### 2. Configure Tailwind

Update `tailwind.config.js`:

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{js,ts,jsx,tsx}", "./src/**/*.html"],
  theme: {
    extend: {
      colors: {
        // SharePoint theme colors
        sharepoint: {
          primary: "#0078d4",
          secondary: "#106ebe",
          accent: "#005a9e",
          neutral: {
            50: "#fafafa",
            100: "#f5f5f5",
            200: "#e5e5e5",
            300: "#d4d4d4",
            400: "#a3a3a3",
            500: "#737373",
            600: "#525252",
            700: "#404040",
            800: "#262626",
            900: "#171717",
          },
        },
        // shadcn/ui compatible colors
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
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      fontFamily: {
        sans: ["Segoe UI", "system-ui", "sans-serif"],
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [
    require("@tailwindcss/typography"),
    require("@tailwindcss/forms"),
    require("@tailwindcss/aspect-ratio"),
  ],
};
```

### 3. Create CSS Variables File

Create `src/styles/globals.css`:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 240 10% 3.9%;
    --card: 0 0% 100%;
    --card-foreground: 240 10% 3.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 240 10% 3.9%;
    --primary: 240 5.9% 10%;
    --primary-foreground: 0 0% 98%;
    --secondary: 240 4.8% 95.9%;
    --secondary-foreground: 240 5.9% 10%;
    --muted: 240 4.8% 95.9%;
    --muted-foreground: 240 3.8% 46.1%;
    --accent: 240 4.8% 95.9%;
    --accent-foreground: 240 5.9% 10%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 0 0% 98%;
    --border: 240 5.9% 90%;
    --input: 240 5.9% 90%;
    --ring: 240 5.9% 10%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 240 10% 3.9%;
    --foreground: 0 0% 98%;
    --card: 240 10% 3.9%;
    --card-foreground: 0 0% 98%;
    --popover: 240 10% 3.9%;
    --popover-foreground: 0 0% 98%;
    --primary: 0 0% 98%;
    --primary-foreground: 240 5.9% 10%;
    --secondary: 240 3.7% 15.9%;
    --secondary-foreground: 0 0% 98%;
    --muted: 240 3.7% 15.9%;
    --muted-foreground: 240 5% 64.9%;
    --accent: 240 3.7% 15.9%;
    --accent-foreground: 0 0% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 0 0% 98%;
    --border: 240 3.7% 15.9%;
    --input: 240 3.7% 15.9%;
    --ring: 240 4.9% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}

/* SharePoint specific overrides */
@layer components {
  .sp-webpart-container {
    @apply bg-transparent;
  }

  .sp-webpart-chrome {
    @apply border-none shadow-none;
  }
}

/* Custom component classes */
@layer components {
  .btn {
    @apply inline-flex items-center justify-center rounded-md text-sm font-medium 
           transition-colors focus-visible:outline-none focus-visible:ring-2 
           focus-visible:ring-ring focus-visible:ring-offset-2 disabled:opacity-50 
           disabled:pointer-events-none ring-offset-background;
  }

  .btn-primary {
    @apply btn bg-primary text-primary-foreground hover:bg-primary/90;
  }

  .btn-secondary {
    @apply btn bg-secondary text-secondary-foreground hover:bg-secondary/80;
  }

  .btn-outline {
    @apply btn border border-input bg-background hover:bg-accent hover:text-accent-foreground;
  }

  .btn-ghost {
    @apply btn hover:bg-accent hover:text-accent-foreground;
  }

  .input {
    @apply flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 
           text-sm ring-offset-background file:border-0 file:bg-transparent file:text-sm 
           file:font-medium placeholder:text-muted-foreground focus-visible:outline-none 
           focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 
           disabled:cursor-not-allowed disabled:opacity-50;
  }

  .card {
    @apply rounded-lg border bg-card text-card-foreground shadow-sm;
  }
}
```

## shadcn/ui Setup

### 1. Initialize shadcn/ui

```bash
npx shadcn@latest init
```

Configure with these options:

- **Framework**: React
- **TypeScript**: Yes
- **Style**: Default
- **Tailwind**: Yes
- **Import alias**: `@/components`

### 2. Update components.json

Create/update `components.json`:

```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "default",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "tailwind.config.js",
    "css": "src/styles/globals.css",
    "baseColor": "slate",
    "cssVariables": true
  },
  "aliases": {
    "components": "src/components",
    "utils": "src/lib/utils"
  }
}
```

### 3. Install Core Components

Install essential shadcn/ui components:

```bash
# Core UI components
npx shadcn@latest add button
npx shadcn@latest add input
npx shadcn@latest add card
npx shadcn@latest add badge
npx shadcn@latest add table
npx shadcn@latest add form
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
npx shadcn@latest add toast
npx shadcn@latest add loading-spinner

# Layout components
npx shadcn@latest add separator
npx shadcn@latest add tabs
npx shadcn@latest add accordion

# Form components
npx shadcn@latest add checkbox
npx shadcn@latest add radio-group
npx shadcn@latest add select
npx shadcn@latest add textarea
npx shadcn@latest add date-picker
```

### 4. Create Utility Functions

Create `src/lib/utils.ts`:

```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export const cn = (...inputs: ClassValue[]) => {
  return twMerge(clsx(inputs));
};

// SharePoint specific utilities
export const spStyles = {
  webPartContainer: "sp-webpart-container",
  webPartChrome: "sp-webpart-chrome",
  fabric: "ms-Fabric",
} as const;

// Color utilities for SharePoint themes
export const getSharePointThemeColors = () => {
  const root = document.documentElement;
  return {
    primary:
      getComputedStyle(root).getPropertyValue("--themePrimary") || "#0078d4",
    secondary:
      getComputedStyle(root).getPropertyValue("--themeSecondary") || "#106ebe",
    tertiary:
      getComputedStyle(root).getPropertyValue("--themeTertiary") || "#005a9e",
  };
};

// Responsive breakpoint utilities
export const breakpoints = {
  sm: "640px",
  md: "768px",
  lg: "1024px",
  xl: "1280px",
  "2xl": "1536px",
} as const;

// Animation utilities
export const animations = {
  fadeIn: "animate-in fade-in duration-200",
  slideIn: "animate-in slide-in-from-bottom-4 duration-300",
  scaleIn: "animate-in zoom-in-95 duration-200",
} as const;
```

## Integration with SPFx

### 1. Import Global Styles

In your main web part file (e.g., `MyAppWebPart.ts`):

```typescript
import "../../../styles/globals.css";

export default class MyAppWebPart extends BaseClientSideWebPart<IMyAppWebPartProps> {
  public render(): void {
    const element: React.ReactElement<IMyAppProps> = React.createElement(
      MyApp,
      {
        description: this.properties.description,
        context: this.context,
        className: cn("w-full min-h-screen bg-background text-foreground"),
      }
    );

    ReactDom.render(element, this.domElement);
  }
}
```

### 2. Component Example with shadcn/ui

Create `src/components/ui/DataTable.tsx`:

```typescript
import React from "react";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Badge } from "@/components/ui/badge";
import { Search, Filter, Plus } from "lucide-react";

type DataItem = {
  id: string;
  title: string;
  status: "active" | "inactive" | "pending";
  createdAt: Date;
};

type DataTableProps = {
  data: DataItem[];
  loading?: boolean;
  onAdd?: () => void;
  onFilter?: (query: string) => void;
  className?: string;
};

export const DataTable: React.FC<DataTableProps> = ({
  data,
  loading = false,
  onAdd,
  onFilter,
  className,
}) => {
  const [searchQuery, setSearchQuery] = React.useState("");

  const handleSearch = (value: string) => {
    setSearchQuery(value);
    onFilter?.(value);
  };

  const getStatusVariant = (status: DataItem["status"]) => {
    switch (status) {
      case "active":
        return "default";
      case "inactive":
        return "secondary";
      case "pending":
        return "outline";
      default:
        return "secondary";
    }
  };

  return (
    <Card className={cn("w-full", className)}>
      <CardHeader>
        <div className="flex items-center justify-between">
          <CardTitle>Data Table</CardTitle>
          {onAdd && (
            <Button onClick={onAdd} className="gap-2">
              <Plus className="h-4 w-4" />
              Add New
            </Button>
          )}
        </div>

        {onFilter && (
          <div className="flex items-center gap-2">
            <div className="relative flex-1">
              <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
              <Input
                placeholder="Search..."
                value={searchQuery}
                onChange={(e) => handleSearch(e.target.value)}
                className="pl-10"
              />
            </div>
            <Button variant="outline" size="icon">
              <Filter className="h-4 w-4" />
            </Button>
          </div>
        )}
      </CardHeader>

      <CardContent>
        {loading ? (
          <div className="flex items-center justify-center p-8">
            <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
          </div>
        ) : (
          <div className="overflow-x-auto">
            <table className="w-full">
              <thead>
                <tr className="border-b">
                  <th className="text-left p-4 font-medium">Title</th>
                  <th className="text-left p-4 font-medium">Status</th>
                  <th className="text-left p-4 font-medium">Created</th>
                </tr>
              </thead>
              <tbody>
                {data.map((item) => (
                  <tr key={item.id} className="border-b hover:bg-muted/50">
                    <td className="p-4">{item.title}</td>
                    <td className="p-4">
                      <Badge variant={getStatusVariant(item.status)}>
                        {item.status}
                      </Badge>
                    </td>
                    <td className="p-4 text-muted-foreground">
                      {item.createdAt.toLocaleDateString()}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </CardContent>
    </Card>
  );
};
```

## SharePoint Theme Integration

### 1. Theme Detection Hook

Create `src/hooks/useSharePointTheme.ts`:

```typescript
import { useEffect, useState } from "react";

type SharePointTheme = {
  primaryColor: string;
  secondaryColor: string;
  backgroundColor: string;
  textColor: string;
  isDark: boolean;
};

export const useSharePointTheme = (): SharePointTheme => {
  const [theme, setTheme] = useState<SharePointTheme>({
    primaryColor: "#0078d4",
    secondaryColor: "#106ebe",
    backgroundColor: "#ffffff",
    textColor: "#323130",
    isDark: false,
  });

  useEffect(() => {
    const updateTheme = () => {
      const root = document.documentElement;
      const computedStyle = getComputedStyle(root);

      const primaryColor =
        computedStyle.getPropertyValue("--themePrimary") || "#0078d4";
      const backgroundColor =
        computedStyle.getPropertyValue("--bodyBackground") || "#ffffff";
      const textColor =
        computedStyle.getPropertyValue("--bodyText") || "#323130";

      // Detect if dark theme
      const isDark = backgroundColor.includes("rgb")
        ? backgroundColor
            .split(",")
            .reduce((sum, val) => sum + parseInt(val.replace(/\D/g, "")), 0) <
          384
        : false;

      setTheme({
        primaryColor,
        secondaryColor:
          computedStyle.getPropertyValue("--themeSecondary") || "#106ebe",
        backgroundColor,
        textColor,
        isDark,
      });
    };

    updateTheme();

    // Watch for theme changes
    const observer = new MutationObserver(updateTheme);
    observer.observe(document.body, {
      attributes: true,
      attributeFilter: ["class", "style"],
    });

    return () => observer.disconnect();
  }, []);

  return theme;
};
```

### 2. Dynamic CSS Variables

Update your main component to apply dynamic theme:

```typescript
import { useSharePointTheme } from "@/hooks/useSharePointTheme";

export const MyApp: React.FC<IMyAppProps> = ({ context }) => {
  const spTheme = useSharePointTheme();

  useEffect(() => {
    // Apply SharePoint theme colors to CSS variables
    const root = document.documentElement;
    root.style.setProperty("--primary", spTheme.primaryColor);
    root.style.setProperty("--background", spTheme.backgroundColor);
    root.style.setProperty("--foreground", spTheme.textColor);
  }, [spTheme]);

  return (
    <div className="min-h-screen bg-background text-foreground">
      {/* Your app content */}
    </div>
  );
};
```

## Build Configuration

### Update gulpfile.js for PostCSS

```javascript
const build = require("@microsoft/sp-build-web");
const path = require("path");

// Configure PostCSS for Tailwind
build.configureWebpack.mergeConfig({
  additionalConfiguration: (generatedConfiguration) => {
    // Find CSS loader rule
    const cssRule = generatedConfiguration.module.rules.find(
      (rule) => rule.test && rule.test.toString().includes("css")
    );

    if (cssRule && cssRule.use) {
      // Add PostCSS loader
      cssRule.use.push({
        loader: "postcss-loader",
        options: {
          postcssOptions: {
            plugins: [require("tailwindcss"), require("autoprefixer")],
          },
        },
      });
    }

    return generatedConfiguration;
  },
});

build.initialize(require("gulp"));
```

## Best Practices

### 1. Component Composition

```typescript
// ✅ Good - Composable components
export const ProjectCard = ({ project, onEdit, onDelete }) => (
  <Card>
    <CardHeader>
      <CardTitle>{project.title}</CardTitle>
    </CardHeader>
    <CardContent>
      <div className="flex justify-between items-center">
        <Badge variant={getStatusVariant(project.status)}>
          {project.status}
        </Badge>
        <div className="flex gap-2">
          <Button variant="outline" size="sm" onClick={() => onEdit(project)}>
            Edit
          </Button>
          <Button
            variant="destructive"
            size="sm"
            onClick={() => onDelete(project)}
          >
            Delete
          </Button>
        </div>
      </div>
    </CardContent>
  </Card>
);
```

### 2. Responsive Design

```typescript
// ✅ Good - Mobile-first responsive design
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
  {projects.map((project) => (
    <ProjectCard key={project.id} project={project} />
  ))}
</div>
```

### 3. Accessibility

```typescript
// ✅ Good - Accessible components
<Button
  variant="primary"
  aria-label={`Edit project ${project.title}`}
  onClick={() => onEdit(project)}
>
  <Edit className="h-4 w-4 mr-2" />
  Edit
</Button>
```

## Next Steps

- [React Functional Patterns](./04-react-functional-patterns.md)
- [Component Architecture](./05-component-architecture.md)
- [SharePoint Lists Integration](./07-sharepoint-lists-integration.md)
