# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a comprehensive documentation repository for building modern SharePoint Framework (SPFx) web parts using React functional components, TypeScript, Tailwind CSS, and shadcn components. The repository contains detailed guides and patterns for SharePoint development.

## Key Commands

### Build and Development
```bash
# Install SPFx tools globally (if not already installed)
npm install -g @microsoft/generator-sharepoint yo gulp-cli

# Build the solution
gulp build

# Start development server
gulp serve

# Clean and rebuild
gulp clean && gulp build

# Package for deployment
gulp bundle --ship
gulp package-solution --ship

# Trust development certificate
gulp trust-dev-cert
```

### Project Creation
```bash
# Create new SPFx project
yo @microsoft/sharepoint
# Select: React, TypeScript, SharePoint Online only (latest)
```

## Architecture & Patterns

### Technology Stack
- **React Functional Components**: Always use `const` declarations with hooks-based architecture
- **TypeScript**: Use `type` definitions instead of `interface` for better flexibility
- **HashRouter**: Essential for SharePoint's routing constraints (never use BrowserRouter)
- **Tailwind CSS**: Utility-first styling that integrates with SharePoint themes
- **shadcn/ui**: Accessible, customizable components
- **PnPjs**: For SharePoint API interactions
- **React Query**: For data fetching and caching

### Component Pattern
```typescript
export const ProjectCard: React.FC<ProjectCardProps> = ({
  project,
  onEdit,
  onDelete,
}) => {
  const [loading, setLoading] = useState(false);
  
  const handleEdit = useCallback(() => {
    onEdit(project.id);
  }, [project.id, onEdit]);
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>{project.title}</CardTitle>
      </CardHeader>
    </Card>
  );
};
```

### TypeScript Types for SharePoint
```typescript
type ProjectItem = SharePointBaseItem & {
  ProjectName: string;
  StartDate: Date;
  EndDate: Date;
  Status: "Planning" | "In Progress" | "Completed" | "On Hold";
  TeamMembers: string[];
};
```

## Important Principles

1. **SharePoint-First Design**: Always consider SharePoint's constraints (throttling, list limits, permissions)
2. **Use SharePoint's Security Model**: Never implement custom authentication
3. **HashRouter is Mandatory**: SharePoint controls server-side routing
4. **Functional Components Only**: No class components
5. **Type Safety**: Enable strict TypeScript settings
6. **Error Handling**: Always handle SharePoint API failures gracefully
7. **Performance**: Consider virtualization for large lists, use memoization strategically

## Documentation Structure

- `00-development-principles.md` - Core principles and reasoning (START HERE)
- `01-project-setup.md` - Initial setup and build configuration
- `02-typescript-configuration.md` - TypeScript patterns
- `03-tailwind-shadcn-setup.md` - Styling setup
- `04-react-functional-patterns.md` - React patterns
- `07-sharepoint-lists-integration.md` - SharePoint Lists as databases
- `08-data-fetching-patterns.md` - React Query patterns
- `14-sharepoint-theme-switching.md` - Theme switching implementation
- `15-analytics-posthog-integration.md` - Analytics integration

## BCM System - Current Project Focus

This repository now contains a complete **Business Capability Model (BCM) system** using Excel Online as the data backend. Key documentation is in the `/BCM-System-Docs/` folder:

- **[00-Table-of-Contents.md](./BCM-System-Docs/00-Table-of-Contents.md)** - 5-page documentation structure
- **[01-BCM-README.md](./BCM-System-Docs/01-BCM-README.md)** - System overview and architecture  
- **[03-BCM-Excel-Setup.md](./BCM-System-Docs/03-BCM-Excel-Setup.md)** - Excel workbook setup guide
- **[04-BCM-Web-Part-Guide.md](./BCM-System-Docs/04-BCM-Web-Part-Guide.md)** - SPFx development guide

### Key Architectural Decisions Made

**✅ Excel Online over SharePoint Lists** - For relational data storage because:
- SharePoint Lists have 5,000 item limits (major constraint for relationships)
- Excel Online provides unlimited relationships with zero maintenance
- Business users can manage data independently using familiar tools
- Bulk editing, import/export capabilities are superior

**✅ Microsoft Graph API integration** - Built into SPFx (MSGraphClientV3), no setup required

**✅ TypeScript `type` definitions** - Preferred over `interface` for flexibility:
```typescript
export type Capability = {
  capabilityName: string;
  parentCapabilityName?: string;
  level: 1 | 2 | 3;
  tier: 'Strategic' | 'Core' | 'Supporting';
};
```

**✅ Zero-maintenance architecture** - Excel Online handles data management, backup, versioning, collaboration

**❌ Avoid Power Automate** - Adds unnecessary complexity for simple data relationships

## Key Considerations

- **Excel Online as the ONLY data storage** (no SharePoint Lists)
- Use Microsoft Graph API (built into SPFx) for Excel integration
- Handle Excel Online sync delays (1-2 minutes for changes to appear)
- Design for business user data management in Excel
- Use PnPjs for SharePoint web part operations (not data storage)
- Use CSS variables to integrate with SharePoint's theming system
- Zero-maintenance architecture is the primary goal