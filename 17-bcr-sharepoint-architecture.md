# BCR SharePoint Lists Architecture

This document provides specific SharePoint List schemas and patterns for implementing a Business Capability Register (BCR) system based on your IA diagram.

## System Overview

Your BCR system manages:
- **Hierarchical Capabilities** (Level 1, 2, 3 with tier structure)
- **Applications** linked to capabilities
- **Linked Applications Grid** for cross-references
- **Audit Controls** and **Processes**
- **Related Business Units**
- **Projects** and integration points

## Core Entity Schemas

### 1. Capabilities List (Master Hierarchy)

```typescript
// src/types/bcr-entities.ts

export type CapabilityEntity = SharePointBaseItem & {
  // Core Fields
  CapabilityCode: string; // Unique identifier (e.g., "CAP001", "CAP001.01")
  CapabilityName: string;
  Description: string;
  
  // Hierarchy Fields
  Level: "Level 1" | "Level 2" | "Level 3";
  Tier: "Tier 1" | "Tier 2" | "Tier 3";
  ParentCapabilityId?: number; // Self-referencing lookup
  HierarchyPath: string; // e.g., "1.2.3" for navigation
  SortOrder: number; // For consistent ordering
  
  // Business Context
  BusinessOwner: SPUser;
  TechnicalOwner?: SPUser;
  BusinessCriticality: "Business Important" | "Business Critical" | "Mission Critical";
  
  // Status and Lifecycle
  Status: "Active" | "Inactive" | "Under Review" | "Deprecated";
  LastReviewDate?: string;
  NextReviewDate?: string;
  
  // Calculated Fields (updated by code)
  TotalApplications?: number;
  TotalSubApplications?: number;
  DirectApplicationCount?: number;
  
  // Metadata
  Category?: string;
  Tags?: string; // Comma-separated for filtering
};

// List Definition
export const capabilitiesListDefinition: RelationalListDefinition = {
  title: "Capabilities",
  description: "Hierarchical business capabilities register",
  template: 100,
  fields: [
    { name: "CapabilityCode", type: "Text", required: true, indexed: true },
    { name: "CapabilityName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    { 
      name: "Level", 
      type: "Choice", 
      choices: ["Level 1", "Level 2", "Level 3"],
      required: true,
      indexed: true 
    },
    { 
      name: "Tier", 
      type: "Choice", 
      choices: ["Tier 1", "Tier 2", "Tier 3"],
      indexed: true 
    },
    { 
      name: "ParentCapability", 
      type: "Lookup", 
      lookupList: "Capabilities",
      lookupField: "CapabilityName",
      indexed: true 
    },
    { name: "HierarchyPath", type: "Text", indexed: true },
    { name: "SortOrder", type: "Number", indexed: true },
    { 
      name: "BusinessOwner", 
      type: "User", 
      required: true,
      indexed: true 
    },
    { name: "TechnicalOwner", type: "User" },
    { 
      name: "BusinessCriticality", 
      type: "Choice", 
      choices: ["Business Important", "Business Critical", "Mission Critical"],
      indexed: true 
    },
    { 
      name: "Status", 
      type: "Choice", 
      choices: ["Active", "Inactive", "Under Review", "Deprecated"],
      indexed: true 
    },
    { name: "LastReviewDate", type: "DateTime" },
    { name: "NextReviewDate", type: "DateTime", indexed: true },
    { name: "TotalApplications", type: "Number" },
    { name: "TotalSubApplications", type: "Number" },
    { name: "DirectApplicationCount", type: "Number" },
    { name: "Category", type: "Text", indexed: true },
    { name: "Tags", type: "Text" },
  ],
  indexes: ["CapabilityCode", "Level", "ParentCapability", "BusinessOwner", "Status"],
  relationships: [
    { type: "one-to-many", relatedList: "Applications", foreignKey: "CapabilityId" },
    { type: "one-to-many", relatedList: "Capabilities", foreignKey: "ParentCapabilityId" },
  ],
};
```

### 2. Applications List

```typescript
export type ApplicationEntity = SharePointBaseItem & {
  // Core Application Info
  ApplicationCode: string; // Unique identifier
  ApplicationName: string;
  Description: string;
  
  // Capability Relationship
  CapabilityId: number; // Primary capability link
  CapabilityPath?: string; // Denormalized for performance
  
  // Application Classification
  TPRIndicator: "Tolerate" | "Invest" | "Migrate" | "Eliminate";
  BusinessCriticality: "Business Important" | "Business Critical" | "Mission Critical";
  
  // Ownership
  ApplicationOwner: SPUser;
  BusinessOwner: SPUser;
  TechnicalOwner?: SPUser;
  VendorProvider?: string;
  
  // Technical Details
  Technology?: string;
  Version?: string;
  HostingModel?: "On-Premise" | "Cloud" | "Hybrid";
  
  // Business Context
  UserCount?: number;
  AnnualCost?: number;
  LicenseType?: string;
  
  // Status and Lifecycle
  Status: "Active" | "Inactive" | "Under Development" | "Being Replaced" | "End of Life";
  GoLiveDate?: string;
  EndOfSupportDate?: string;
  
  // Analysis Fields
  TimeAnalysisScore?: number;
  TPRAnalysisStatus?: "Not Started" | "In Progress" | "Complete";
  
  // Integration tracking
  IntegrationCount?: number;
  IsSupplierApplication?: boolean;
  
  // Metadata
  Tags?: string;
  Notes?: string;
};

export const applicationsListDefinition: RelationalListDefinition = {
  title: "Applications",
  description: "Application registry linked to capabilities",
  template: 100,
  fields: [
    { name: "ApplicationCode", type: "Text", required: true, indexed: true },
    { name: "ApplicationName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    { 
      name: "Capability", 
      type: "Lookup", 
      required: true,
      lookupList: "Capabilities",
      lookupField: "CapabilityName",
      indexed: true 
    },
    { name: "CapabilityPath", type: "Text", indexed: true },
    { 
      name: "TPRIndicator", 
      type: "Choice", 
      choices: ["Tolerate", "Invest", "Migrate", "Eliminate"],
      indexed: true 
    },
    { 
      name: "BusinessCriticality", 
      type: "Choice", 
      choices: ["Business Important", "Business Critical", "Mission Critical"],
      indexed: true 
    },
    { name: "ApplicationOwner", type: "User", required: true, indexed: true },
    { name: "BusinessOwner", type: "User", required: true, indexed: true },
    { name: "TechnicalOwner", type: "User" },
    { name: "VendorProvider", type: "Text" },
    { name: "Technology", type: "Text", indexed: true },
    { name: "Version", type: "Text" },
    { 
      name: "HostingModel", 
      type: "Choice", 
      choices: ["On-Premise", "Cloud", "Hybrid"],
      indexed: true 
    },
    { name: "UserCount", type: "Number" },
    { name: "AnnualCost", type: "Number" },
    { name: "LicenseType", type: "Text" },
    { 
      name: "Status", 
      type: "Choice", 
      choices: ["Active", "Inactive", "Under Development", "Being Replaced", "End of Life"],
      indexed: true 
    },
    { name: "GoLiveDate", type: "DateTime" },
    { name: "EndOfSupportDate", type: "DateTime", indexed: true },
    { name: "TimeAnalysisScore", type: "Number" },
    { 
      name: "TPRAnalysisStatus", 
      type: "Choice", 
      choices: ["Not Started", "In Progress", "Complete"] 
    },
    { name: "IntegrationCount", type: "Number" },
    { name: "IsSupplierApplication", type: "Boolean" },
    { name: "Tags", type: "Text" },
    { name: "Notes", type: "Text" },
  ],
  indexes: ["ApplicationCode", "Capability", "TPRIndicator", "BusinessCriticality", "Status"],
  relationships: [
    { type: "many-to-one", relatedList: "Capabilities", foreignKey: "CapabilityId" },
    { type: "one-to-many", relatedList: "LinkedApplications", foreignKey: "SourceApplicationId" },
    { type: "one-to-many", relatedList: "ApplicationIntegrations", foreignKey: "ApplicationId" },
  ],
};
```

### 3. Linked Applications (Many-to-Many Relationships)

```typescript
export type LinkedApplicationEntity = SharePointBaseItem & {
  SourceApplicationId: number;
  TargetApplicationId: number;
  RelationshipType: "Integration" | "Data Flow" | "Dependency" | "Replacement" | "Supplier Application";
  IntegrationName?: string;
  IntegrationType?: "API" | "File Transfer" | "Database" | "Message Queue" | "Real-time" | "Batch";
  DataFlowDirection?: "Bidirectional" | "Source to Target" | "Target to Source";
  Criticality: "Low" | "Medium" | "High" | "Critical";
  Status: "Active" | "Inactive" | "Planned" | "Under Development";
  Description?: string;
  LastReviewDate?: string;
  Notes?: string;
};

export const linkedApplicationsListDefinition: RelationalListDefinition = {
  title: "LinkedApplications", 
  description: "Many-to-many relationships between applications",
  template: 100,
  fields: [
    { 
      name: "SourceApplication", 
      type: "Lookup", 
      required: true,
      lookupList: "Applications",
      lookupField: "ApplicationName",
      indexed: true 
    },
    { 
      name: "TargetApplication", 
      type: "Lookup", 
      required: true,
      lookupList: "Applications",
      lookupField: "ApplicationName",
      indexed: true 
    },
    { 
      name: "RelationshipType", 
      type: "Choice", 
      choices: ["Integration", "Data Flow", "Dependency", "Replacement", "Supplier Application"],
      required: true,
      indexed: true 
    },
    { name: "IntegrationName", type: "Text" },
    { 
      name: "IntegrationType", 
      type: "Choice", 
      choices: ["API", "File Transfer", "Database", "Message Queue", "Real-time", "Batch"] 
    },
    { 
      name: "DataFlowDirection", 
      type: "Choice", 
      choices: ["Bidirectional", "Source to Target", "Target to Source"] 
    },
    { 
      name: "Criticality", 
      type: "Choice", 
      choices: ["Low", "Medium", "High", "Critical"],
      indexed: true 
    },
    { 
      name: "Status", 
      type: "Choice", 
      choices: ["Active", "Inactive", "Planned", "Under Development"],
      indexed: true 
    },
    { name: "Description", type: "Text" },
    { name: "LastReviewDate", type: "DateTime" },
    { name: "Notes", type: "Text" },
  ],
  indexes: ["SourceApplication", "TargetApplication", "RelationshipType", "Status"],
  relationships: [
    { type: "many-to-one", relatedList: "Applications", foreignKey: "SourceApplicationId" },
    { type: "many-to-one", relatedList: "Applications", foreignKey: "TargetApplicationId" },
  ],
};
```

### 4. Business Units Grid

```typescript
export type BusinessUnitEntity = SharePointBaseItem & {
  BusinessUnitCode: string;
  BusinessUnitName: string;
  Description?: string;
  BusinessUnitHead: SPUser;
  ParentBusinessUnitId?: number;
  Level: "Division" | "Department" | "Team";
  CostCenter?: string;
  Location?: string;
  IsActive: boolean;
  
  // Calculated fields
  ApplicationCount?: number;
  CapabilityCount?: number;
};

export const businessUnitsListDefinition: RelationalListDefinition = {
  title: "BusinessUnits",
  description: "Organizational business units",
  template: 100,
  fields: [
    { name: "BusinessUnitCode", type: "Text", required: true, indexed: true },
    { name: "BusinessUnitName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    { name: "BusinessUnitHead", type: "User", required: true },
    { 
      name: "ParentBusinessUnit", 
      type: "Lookup", 
      lookupList: "BusinessUnits",
      lookupField: "BusinessUnitName" 
    },
    { 
      name: "Level", 
      type: "Choice", 
      choices: ["Division", "Department", "Team"],
      indexed: true 
    },
    { name: "CostCenter", type: "Text" },
    { name: "Location", type: "Text" },
    { name: "IsActive", type: "Boolean", indexed: true },
    { name: "ApplicationCount", type: "Number" },
    { name: "CapabilityCount", type: "Number" },
  ],
  indexes: ["BusinessUnitCode", "Level", "IsActive"],
  relationships: [
    { type: "many-to-many", relatedList: "Capabilities", junctionList: "CapabilityBusinessUnits" },
    { type: "many-to-many", relatedList: "Applications", junctionList: "ApplicationBusinessUnits" },
  ],
};
```

### 5. Projects and Processes

```typescript
export type ProjectEntity = SharePointBaseItem & {
  ProjectCode: string;
  ProjectName: string;
  Description: string;
  ProjectManager: SPUser;
  Status: "Planning" | "Active" | "On Hold" | "Completed" | "Cancelled";
  StartDate: string;
  EndDate?: string;
  Budget?: number;
  
  // BCR Context
  ImpactedCapabilities?: string; // Comma-separated capability IDs
  ImpactedApplications?: string; // Comma-separated application IDs
  ProjectType: "New Capability" | "Application Replacement" | "Integration" | "Process Improvement";
};

export type ProcessEntity = SharePointBaseItem & {
  ProcessCode: string;
  ProcessName: string;
  Description: string;
  ProcessOwner: SPUser;
  
  // Linked Capabilities
  LinkedCapabilities?: string; // Comma-separated capability IDs
  
  // Process Classification
  ProcessType: "Core" | "Support" | "Management";
  Criticality: "Low" | "Medium" | "High" | "Critical";
  
  // Audit Control Links
  HasAuditControls: boolean;
  AuditControlCount?: number;
};

export type AuditControlEntity = SharePointBaseItem & {
  ControlCode: string;
  ControlName: string;
  Description: string;
  ControlOwner: SPUser;
  
  // Process Links
  ProcessId?: number;
  
  // Control Classification
  ControlType: "Preventive" | "Detective" | "Corrective";
  RiskLevel: "Low" | "Medium" | "High" | "Critical";
  
  // Status
  Status: "Active" | "Inactive" | "Under Review";
  LastTestDate?: string;
  NextTestDate?: string;
  
  // Filters for your system
  FilterCriteria?: string;
};
```

## Hierarchical Navigation Patterns

### 1. Capability Hierarchy Service

```typescript
// src/lib/bcr-hierarchy-service.ts
export class BCRHierarchyService extends SharePointService {
  
  // Build complete capability tree
  async getCapabilityTree(): Promise<CapabilityTreeNode[]> {
    const allCapabilities = await this.getListItems<CapabilityEntity>(
      "Capabilities",
      ["*", "BusinessOwner/Title", "BusinessOwner/Email", "ParentCapability/Title"],
      ["BusinessOwner", "ParentCapability"],
      "Status eq 'Active'",
      "Level,SortOrder"
    );
    
    return this.buildHierarchyTree(allCapabilities);
  }
  
  private buildHierarchyTree(capabilities: CapabilityEntity[]): CapabilityTreeNode[] {
    const nodeMap = new Map<number, CapabilityTreeNode>();
    const rootNodes: CapabilityTreeNode[] = [];
    
    // Create all nodes
    capabilities.forEach(cap => {
      nodeMap.set(cap.Id, {
        ...cap,
        children: [],
        applicationCount: cap.DirectApplicationCount || 0,
        totalApplicationCount: cap.TotalApplications || 0,
      });
    });
    
    // Build parent-child relationships
    capabilities.forEach(cap => {
      const node = nodeMap.get(cap.Id)!;
      
      if (cap.ParentCapabilityId) {
        const parent = nodeMap.get(cap.ParentCapabilityId);
        if (parent) {
          parent.children.push(node);
        }
      } else {
        rootNodes.push(node);
      }
    });
    
    return rootNodes;
  }
  
  // Get capability with all applications
  async getCapabilityWithApplications(capabilityId: number): Promise<{
    capability: CapabilityEntity;
    directApplications: ApplicationEntity[];
    allApplications: ApplicationEntity[]; // Including child capabilities
    childCapabilities: CapabilityEntity[];
    linkedApplications: LinkedApplicationEntity[];
  }> {
    
    const [capability, directApplications, childCapabilities] = await Promise.all([
      this.getListItemById<CapabilityEntity>("Capabilities", capabilityId),
      this.getListItems<ApplicationEntity>(
        "Applications",
        ["*", "ApplicationOwner/Title", "BusinessOwner/Title"],
        ["ApplicationOwner", "BusinessOwner"],
        `Capability/Id eq ${capabilityId}`
      ),
      this.getListItems<CapabilityEntity>(
        "Capabilities", 
        ["*"],
        [],
        `ParentCapability/Id eq ${capabilityId}`
      ),
    ]);
    
    // Get applications from child capabilities
    let allApplications = [...directApplications];
    
    if (childCapabilities.length > 0) {
      const childCapabilityIds = childCapabilities.map(c => c.Id);
      const childApplicationsPromises = childCapabilityIds.map(id =>
        this.getListItems<ApplicationEntity>(
          "Applications",
          ["*", "ApplicationOwner/Title", "BusinessOwner/Title"],
          ["ApplicationOwner", "BusinessOwner"],
          `Capability/Id eq ${id}`
        )
      );
      
      const childApplicationsResults = await Promise.all(childApplicationsPromises);
      childApplicationsResults.forEach(apps => {
        allApplications = [...allApplications, ...apps];
      });
    }
    
    // Get linked applications
    const applicationIds = allApplications.map(app => app.Id);
    let linkedApplications: LinkedApplicationEntity[] = [];
    
    if (applicationIds.length > 0) {
      const sourceLinks = await this.getListItems<LinkedApplicationEntity>(
        "LinkedApplications",
        ["*", "SourceApplication/Title", "TargetApplication/Title"],
        ["SourceApplication", "TargetApplication"],
        applicationIds.map(id => `SourceApplication/Id eq ${id}`).join(" or ")
      );
      
      const targetLinks = await this.getListItems<LinkedApplicationEntity>(
        "LinkedApplications",
        ["*", "SourceApplication/Title", "TargetApplication/Title"],
        ["SourceApplication", "TargetApplication"],
        applicationIds.map(id => `TargetApplication/Id eq ${id}`).join(" or ")
      );
      
      linkedApplications = [...sourceLinks, ...targetLinks];
    }
    
    return {
      capability,
      directApplications,
      allApplications,
      childCapabilities,
      linkedApplications,
    };
  }
  
  // Update calculated fields for capability hierarchy
  async updateCapabilityCalculatedFields(capabilityId: number): Promise<void> {
    const { directApplications, allApplications, childCapabilities } = 
      await this.getCapabilityWithApplications(capabilityId);
    
    const calculatedFields = {
      DirectApplicationCount: directApplications.length,
      TotalApplications: allApplications.length,
      TotalSubApplications: allApplications.length - directApplications.length,
    };
    
    await this.updateListItem("Capabilities", capabilityId, calculatedFields);
    
    // Update parent capabilities recursively
    const capability = await this.getListItemById<CapabilityEntity>("Capabilities", capabilityId);
    if (capability.ParentCapabilityId) {
      await this.updateCapabilityCalculatedFields(capability.ParentCapabilityId);
    }
  }
}

type CapabilityTreeNode = CapabilityEntity & {
  children: CapabilityTreeNode[];
  applicationCount: number;
  totalApplicationCount: number;
};
```

## Import Mappings for BCR Data

### 1. Capability Import Mapping

```typescript
// src/lib/bcr-import-mappings.ts
export const capabilityImportMapping: ImportMapping[] = [
  {
    sourceColumn: "Capability Code",
    targetField: "CapabilityCode",
    fieldType: "Text",
    required: true,
    validation: (value: string) => /^CAP\d{3}(\.\d{2})*$/.test(value), // CAP001 or CAP001.01
  },
  {
    sourceColumn: "Capability Name",
    targetField: "Title",
    fieldType: "Text",
    required: true,
  },
  {
    sourceColumn: "Description",
    targetField: "Description",
    fieldType: "Text",
    required: false,
  },
  {
    sourceColumn: "Level",
    targetField: "Level",
    fieldType: "Choice",
    required: true,
    transform: (value: any) => {
      const levelMap: { [key: string]: string } = {
        "1": "Level 1",
        "2": "Level 2", 
        "3": "Level 3",
        "level 1": "Level 1",
        "level 2": "Level 2",
        "level 3": "Level 3",
      };
      return levelMap[String(value).toLowerCase()] || value;
    },
  },
  {
    sourceColumn: "Tier",
    targetField: "Tier",
    fieldType: "Choice",
    required: false,
    transform: (value: any) => {
      const tierMap: { [key: string]: string } = {
        "1": "Tier 1",
        "2": "Tier 2",
        "3": "Tier 3",
        "tier 1": "Tier 1",
        "tier 2": "Tier 2", 
        "tier 3": "Tier 3",
      };
      return tierMap[String(value).toLowerCase()] || value;
    },
  },
  {
    sourceColumn: "Parent Capability Code",
    targetField: "ParentCapability",
    fieldType: "Lookup",
    required: false,
    lookupList: "Capabilities",
    lookupField: "CapabilityCode",
  },
  {
    sourceColumn: "Business Owner Email",
    targetField: "BusinessOwner",
    fieldType: "User",
    required: true,
  },
  {
    sourceColumn: "Business Criticality",
    targetField: "BusinessCriticality",
    fieldType: "Choice",
    required: false,
    transform: (value: any) => {
      const criticalityMap: { [key: string]: string } = {
        "important": "Business Important",
        "critical": "Business Critical",
        "mission critical": "Mission Critical",
        "business important": "Business Important",
        "business critical": "Business Critical",
      };
      return criticalityMap[String(value).toLowerCase()] || "Business Important";
    },
  },
];

export const applicationImportMapping: ImportMapping[] = [
  {
    sourceColumn: "Application Code",
    targetField: "ApplicationCode", 
    fieldType: "Text",
    required: true,
    validation: (value: string) => /^APP\d{3,6}$/.test(value), // APP001 to APP999999
  },
  {
    sourceColumn: "Application Name",
    targetField: "Title",
    fieldType: "Text",
    required: true,
  },
  {
    sourceColumn: "Capability Code",
    targetField: "Capability",
    fieldType: "Lookup",
    required: true,
    lookupList: "Capabilities",
    lookupField: "CapabilityCode",
  },
  {
    sourceColumn: "TPR Indicator", 
    targetField: "TPRIndicator",
    fieldType: "Choice",
    required: false,
    transform: (value: any) => {
      const tprMap: { [key: string]: string } = {
        "t": "Tolerate",
        "i": "Invest",
        "m": "Migrate", 
        "e": "Eliminate",
        "tolerate": "Tolerate",
        "invest": "Invest",
        "migrate": "Migrate",
        "eliminate": "Eliminate",
      };
      return tprMap[String(value).toLowerCase()] || value;
    },
  },
  {
    sourceColumn: "Application Owner Email",
    targetField: "ApplicationOwner",
    fieldType: "User", 
    required: true,
  },
  {
    sourceColumn: "Business Owner Email",
    targetField: "BusinessOwner",
    fieldType: "User",
    required: true,
  },
  {
    sourceColumn: "Vendor/Provider",
    targetField: "VendorProvider",
    fieldType: "Text",
    required: false,
  },
  {
    sourceColumn: "Annual Cost",
    targetField: "AnnualCost",
    fieldType: "Number",
    required: false,
    transform: (value: any) => {
      // Remove currency symbols and commas
      const cleaned = String(value).replace(/[$,£€]/g, '');
      return Number(cleaned) || 0;
    },
  },
];
```

## Usage Patterns for Your BCR System

### 1. Navigation Component

```typescript
// src/components/BCRNavigation.tsx
export const BCRNavigation: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const [capabilityTree, setCapabilityTree] = useState<CapabilityTreeNode[]>([]);
  const [selectedCapability, setSelectedCapability] = useState<number | null>(null);
  
  const hierarchyService = useMemo(() => new BCRHierarchyService(context), [context]);
  
  useEffect(() => {
    const loadCapabilities = async () => {
      try {
        const tree = await hierarchyService.getCapabilityTree();
        setCapabilityTree(tree);
      } catch (error) {
        console.error('Failed to load capabilities:', error);
      }
    };
    
    loadCapabilities();
  }, [hierarchyService]);
  
  const renderCapabilityNode = (node: CapabilityTreeNode, level: number = 0) => (
    <div key={node.Id} className={`ml-${level * 4}`}>
      <div 
        className="flex items-center p-2 hover:bg-gray-100 cursor-pointer"
        onClick={() => setSelectedCapability(node.Id)}
      >
        <span className="text-sm text-gray-500 mr-2">{node.CapabilityCode}</span>
        <span className="font-medium">{node.Title}</span>
        <span className="ml-auto text-xs text-gray-400">
          {node.applicationCount} apps
        </span>
      </div>
      {node.children.map(child => renderCapabilityNode(child, level + 1))}
    </div>
  );
  
  return (
    <div className="w-full">
      <div className="border-r h-full overflow-y-auto">
        {capabilityTree.map(node => renderCapabilityNode(node))}
      </div>
    </div>
  );
};
```

This architecture provides a complete SharePoint Lists implementation for your BCR system with proper hierarchical relationships, import capabilities, and navigation patterns!

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Write comprehensive SharePoint Lists relational database guidance", "status": "completed", "priority": "high"}, {"id": "2", "content": "Review IA image when provided by user", "status": "completed", "priority": "high"}, {"id": "3", "content": "Create guidance for importing spreadsheets to SharePoint Lists", "status": "completed", "priority": "high"}, {"id": "4", "content": "Update existing SharePoint lists documentation with relational patterns", "status": "pending", "priority": "medium"}, {"id": "5", "content": "Create BCR-specific SharePoint list schema based on IA diagram", "status": "completed", "priority": "high"}, {"id": "6", "content": "Design hierarchical capability structure for SharePoint Lists", "status": "completed", "priority": "high"}]