# Complete Relationship Guide: Excel → SharePoint → React

This guide provides step-by-step instructions for implementing one-to-many and many-to-many relationships from Excel preparation through SharePoint import to React web part queries.

## Overview of Relationships We'll Build

### One-to-Many Relationships:
1. **Capability → Applications** (One capability has many applications)
2. **Capability → Sub-Capabilities** (One parent capability has many child capabilities)

### Many-to-Many Relationships:
3. **Applications ↔ Capabilities** (Applications can support multiple capabilities, capabilities can be supported by multiple applications)

## Part 1: Excel Data Preparation with Power Query

### Step 1: Create Base Data Tables

**Create a new Excel workbook: `BCR-Data.xlsx`**

**Sheet 1: "Capabilities"**
```
CapabilityCode | CapabilityName              | Level    | BusinessArea        | ParentCapabilityCode
CAP001        | Customer Management          | Level 1  | Customer           | 
CAP001.01     | Customer Registration        | Level 2  | Customer           | CAP001
CAP001.02     | Customer Support            | Level 2  | Customer           | CAP001
CAP001.03     | Customer Analytics          | Level 2  | Customer           | CAP001
CAP002        | Finance Management          | Level 1  | Finance            | 
CAP002.01     | Accounts Payable            | Level 2  | Finance            | CAP002
CAP002.02     | Financial Reporting         | Level 2  | Finance            | CAP002
CAP003        | Shared Services             | Level 1  | Shared             | 
CAP003.01     | Authentication              | Level 2  | Shared             | CAP003
CAP003.02     | Data Storage                | Level 2  | Shared             | CAP003
```

**Sheet 2: "Applications"**
```
ApplicationCode | ApplicationName        | PrimaryBusinessArea | SecondaryBusinessArea | ApplicationOwner
APP001         | Customer Portal        | Customer           | Shared               | john@company.com
APP002         | CRM System            | Customer           |                      | jane@company.com
APP003         | Finance Dashboard     | Finance            | Shared               | bob@company.com
APP004         | ERP System            | Finance            | Customer             | alice@company.com
APP005         | Identity Provider     | Shared             |                      | charlie@company.com
APP006         | Data Warehouse        | Shared             | Finance              | diana@company.com
```

### Step 2: Create Relationship Rules Table

**Sheet 3: "RelationshipRules"**
```
RuleName                    | SourceType    | SourceFilter              | TargetType    | TargetFilter                    | RelationshipType
Primary Business Area Match | Application   | PrimaryBusinessArea       | Capability    | BusinessArea,Level=Level 2      | Primary
Secondary Area Support      | Application   | SecondaryBusinessArea     | Capability    | BusinessArea,Level=Level 2      | Supporting
Shared Service Usage        | Application   | *                         | Capability    | BusinessArea=Shared,Level=Level 2| Supporting
```

### Step 3: Generate One-to-Many Relationships with Power Query

**Create Capability Hierarchy (Parent-Child):**

1. **Data** → **Get Data** → **From Other Sources** → **Blank Query**
2. **Advanced Editor** → Paste this M code:

```m
let
    // Get capabilities table
    Source = Excel.CurrentWorkbook(){[Name="Capabilities"]}[Content],
    
    // Filter only records that have a parent
    HasParent = Table.SelectRows(Source, each [ParentCapabilityCode] <> null and [ParentCapabilityCode] <> ""),
    
    // Create the hierarchy junction table
    CapabilityHierarchy = Table.SelectColumns(HasParent, {
        "ParentCapabilityCode", 
        "CapabilityCode", 
        "Level"
    }),
    
    // Add relationship metadata
    AddRelationType = Table.AddColumn(CapabilityHierarchy, "RelationshipType", each "Parent-Child"),
    AddSortOrder = Table.AddIndexColumn(AddRelationType, "SortOrder", 1, 1),
    AddIsActive = Table.AddColumn(AddSortOrder, "IsActive", each true),
    
    // Rename columns for junction table
    RenameColumns = Table.RenameColumns(AddIsActive, {
        {"CapabilityCode", "ChildCapabilityCode"},
        {"Level", "ChildLevel"}
    })
in
    RenameColumns
```

3. **Close & Load** → Name this query **"CapabilityHierarchy"**

### Step 4: Generate Many-to-Many Relationships with Power Query

**Create Application-Capability Relationships:**

1. **Data** → **Get Data** → **From Other Sources** → **Blank Query**
2. **Advanced Editor** → Paste this M code:

```m
let
    // Get source tables
    Applications = Excel.CurrentWorkbook(){[Name="Applications"]}[Content],
    Capabilities = Excel.CurrentWorkbook(){[Name="Capabilities"]}[Content],
    Rules = Excel.CurrentWorkbook(){[Name="RelationshipRules"]}[Content],
    
    // Function to apply a single rule
    ApplyRule = (rule as record) =>
        let
            // Get matching applications based on rule
            MatchingApps = if rule[SourceFilter] = "*" 
                then Applications
                else if Text.Contains(rule[SourceFilter], "PrimaryBusinessArea")
                    then Table.SelectRows(Applications, each [PrimaryBusinessArea] <> null and [PrimaryBusinessArea] <> "")
                else if Text.Contains(rule[SourceFilter], "SecondaryBusinessArea") 
                    then Table.SelectRows(Applications, each [SecondaryBusinessArea] <> null and [SecondaryBusinessArea] <> "")
                else Applications,
            
            // Create relationships for each matching app
            CreateRelationships = Table.ExpandTableColumn(
                Table.AddColumn(MatchingApps, "MatchingCapabilities", (app) =>
                    let
                        // Determine which business area to match against
                        BusinessAreaToMatch = if Text.Contains(rule[SourceFilter], "PrimaryBusinessArea")
                            then app[PrimaryBusinessArea]
                        else if Text.Contains(rule[SourceFilter], "SecondaryBusinessArea")
                            then app[SecondaryBusinessArea]  
                        else "Shared",
                        
                        // Find matching capabilities
                        MatchingCaps = Table.SelectRows(Capabilities, 
                            each [BusinessArea] = BusinessAreaToMatch and
                                 (if Text.Contains(rule[TargetFilter], "Level=Level 2") then [Level] = "Level 2" else true)
                        )
                    in
                        MatchingCaps
                ), 
                "MatchingCapabilities", 
                {"CapabilityCode", "CapabilityName", "BusinessArea"}, 
                {"CapabilityCode", "CapabilityName", "CapabilityBusinessArea"}
            ),
            
            // Add rule metadata
            AddRuleInfo = Table.AddColumn(CreateRelationships, "RelationshipType", each rule[RelationshipType]),
            AddRuleName = Table.AddColumn(AddRuleInfo, "RuleName", each rule[RuleName])
        in
            AddRuleInfo,
    
    // Apply all rules
    AllRelationships = Table.Combine(List.Transform(Table.ToRecords(Rules), ApplyRule)),
    
    // Clean up final table
    FinalTable = Table.SelectColumns(AllRelationships, {
        "ApplicationCode", 
        "CapabilityCode", 
        "RelationshipType",
        "RuleName"
    }),
    
    // Add metadata
    AddIsActive = Table.AddColumn(FinalTable, "IsActive", each true),
    AddCreatedDate = Table.AddColumn(AddIsActive, "CreatedDate", each DateTime.LocalNow())
in
    AddCreatedDate
```

3. **Close & Load** → Name this query **"ApplicationCapabilities"**

### Step 5: Export Junction Tables

You should now have 3 sheets in your workbook:
- **Capabilities** (original data)
- **Applications** (original data)  
- **CapabilityHierarchy** (generated one-to-many)
- **ApplicationCapabilities** (generated many-to-many)

**Save each as separate CSV files for SharePoint import:**
- `Capabilities.csv`
- `Applications.csv`
- `CapabilityHierarchy.csv`
- `ApplicationCapabilities.csv`

## Part 2: SharePoint Lists Setup

### Step 1: Create Base Entity Lists

**Create these lists manually in SharePoint (in this order):**

**List 1: "Capabilities"**
```
Column Name          | Type    | Required | Indexed
Title               | Text    | Yes      | Yes
CapabilityCode      | Text    | Yes      | Yes  
Level               | Choice  | Yes      | Yes
BusinessArea        | Choice  | No       | Yes
```

**List 2: "Applications"**  
```
Column Name          | Type    | Required | Indexed
Title               | Text    | Yes      | Yes
ApplicationCode     | Text    | Yes      | Yes
PrimaryBusinessArea | Choice  | No       | Yes  
ApplicationOwner    | Person  | Yes      | Yes
```

### Step 2: Create Junction Lists

**List 3: "CapabilityHierarchy"**
```
Column Name             | Type    | Required | Indexed
Title                  | Text    | Yes      | Yes
ParentCapabilityCode   | Text    | Yes      | Yes
ChildCapabilityCode    | Text    | Yes      | Yes
RelationshipType       | Choice  | Yes      | No
SortOrder             | Number  | No       | No
IsActive              | Boolean | No       | Yes
```

**List 4: "ApplicationCapabilities"**
```
Column Name         | Type    | Required | Indexed
Title              | Text    | Yes      | Yes
ApplicationCode    | Text    | Yes      | Yes
CapabilityCode     | Text    | Yes      | Yes
RelationshipType   | Choice  | Yes      | Yes
IsActive          | Boolean | No       | Yes
```

### Step 3: Import CSV Data

**Import each CSV file using SharePoint's "Import from spreadsheet" feature:**

1. Go to each list → **New** → **Import from spreadsheet**
2. Upload the corresponding CSV file
3. Map columns correctly
4. Import data

### Step 4: Add Lookup Fields (After Import)

**Add lookup fields to resolve relationships:**

**In CapabilityHierarchy list:**
- Add **ParentCapability** lookup field → Points to Capabilities list, shows CapabilityCode
- Add **ChildCapability** lookup field → Points to Capabilities list, shows CapabilityCode

**In ApplicationCapabilities list:**
- Add **Application** lookup field → Points to Applications list, shows ApplicationCode  
- Add **Capability** lookup field → Points to Capabilities list, shows CapabilityCode

## Part 3: React SharePoint Web Part Queries

### Step 1: Create Type Definitions

```typescript
// src/types/bcr-relationships.ts

export type CapabilityEntity = SharePointBaseItem & {
  CapabilityCode: string;
  Level: "Level 1" | "Level 2" | "Level 3";
  BusinessArea: string;
};

export type ApplicationEntity = SharePointBaseItem & {
  ApplicationCode: string;
  PrimaryBusinessArea: string;
  ApplicationOwner: SPUser;
};

export type CapabilityHierarchyJunction = SharePointBaseItem & {
  ParentCapabilityCode: string;
  ChildCapabilityCode: string;
  RelationshipType: string;
  SortOrder: number;
  IsActive: boolean;
  // Lookup fields (resolved by SharePoint)
  ParentCapability?: SPLookupValue;
  ChildCapability?: SPLookupValue;
};

export type ApplicationCapabilityJunction = SharePointBaseItem & {
  ApplicationCode: string;
  CapabilityCode: string;
  RelationshipType: "Primary" | "Supporting";
  IsActive: boolean;
  // Lookup fields (resolved by SharePoint)
  Application?: SPLookupValue;
  Capability?: SPLookupValue;
};
```

### Step 2: Create Relationship Query Service

```typescript
// src/lib/bcr-relationship-service.ts
export class BCRRelationshipService extends SharePointService {
  
  // ONE-TO-MANY: Get all applications for a capability
  async getApplicationsForCapability(capabilityCode: string): Promise<{
    capability: CapabilityEntity;
    applications: ApplicationEntity[];
    relationships: ApplicationCapabilityJunction[];
  }> {
    
    // Get the capability
    const capabilities = await this.getListItems<CapabilityEntity>(
      "Capabilities",
      ["*"],
      [],
      `CapabilityCode eq '${capabilityCode}'`
    );
    
    if (capabilities.length === 0) {
      throw new Error(`Capability ${capabilityCode} not found`);
    }
    
    const capability = capabilities[0];
    
    // Get relationship junctions for this capability
    const relationships = await this.getListItems<ApplicationCapabilityJunction>(
      "ApplicationCapabilities",
      ["*", "Application/Title", "Application/ApplicationCode", "Capability/Title"],
      ["Application", "Capability"],
      `CapabilityCode eq '${capabilityCode}' and IsActive eq true`
    );
    
    // Get the applications through the junction
    const applicationCodes = relationships.map(rel => rel.ApplicationCode);
    
    const applications = applicationCodes.length > 0 
      ? await this.getListItems<ApplicationEntity>(
          "Applications",
          ["*", "ApplicationOwner/Title", "ApplicationOwner/Email"],
          ["ApplicationOwner"],
          applicationCodes.map(code => `ApplicationCode eq '${code}'`).join(" or ")
        )
      : [];
    
    return {
      capability,
      applications,
      relationships
    };
  }
  
  // ONE-TO-MANY: Get all sub-capabilities for a parent capability
  async getSubCapabilities(parentCapabilityCode: string): Promise<{
    parentCapability: CapabilityEntity;
    childCapabilities: CapabilityEntity[];
    hierarchyRelationships: CapabilityHierarchyJunction[];
  }> {
    
    // Get parent capability
    const parentCapabilities = await this.getListItems<CapabilityEntity>(
      "Capabilities",
      ["*"],
      [],
      `CapabilityCode eq '${parentCapabilityCode}'`
    );
    
    if (parentCapabilities.length === 0) {
      throw new Error(`Parent capability ${parentCapabilityCode} not found`);
    }
    
    const parentCapability = parentCapabilities[0];
    
    // Get hierarchy relationships
    const hierarchyRelationships = await this.getListItems<CapabilityHierarchyJunction>(
      "CapabilityHierarchy",
      ["*", "ParentCapability/Title", "ChildCapability/Title"],
      ["ParentCapability", "ChildCapability"],
      `ParentCapabilityCode eq '${parentCapabilityCode}' and IsActive eq true`,
      "SortOrder"
    );
    
    // Get child capabilities
    const childCapabilityCodes = hierarchyRelationships.map(rel => rel.ChildCapabilityCode);
    
    const childCapabilities = childCapabilityCodes.length > 0
      ? await this.getListItems<CapabilityEntity>(
          "Capabilities",
          ["*"],
          [],
          childCapabilityCodes.map(code => `CapabilityCode eq '${code}'`).join(" or ")
        )
      : [];
    
    return {
      parentCapability,
      childCapabilities,
      hierarchyRelationships
    };
  }
  
  // MANY-TO-MANY: Get all capabilities for an application
  async getCapabilitiesForApplication(applicationCode: string): Promise<{
    application: ApplicationEntity;
    capabilities: CapabilityEntity[];
    relationships: ApplicationCapabilityJunction[];
    primaryCapabilities: CapabilityEntity[];
    supportingCapabilities: CapabilityEntity[];
  }> {
    
    // Get the application
    const applications = await this.getListItems<ApplicationEntity>(
      "Applications",
      ["*", "ApplicationOwner/Title", "ApplicationOwner/Email"],
      ["ApplicationOwner"],
      `ApplicationCode eq '${applicationCode}'`
    );
    
    if (applications.length === 0) {
      throw new Error(`Application ${applicationCode} not found`);
    }
    
    const application = applications[0];
    
    // Get all relationships for this application
    const relationships = await this.getListItems<ApplicationCapabilityJunction>(
      "ApplicationCapabilities",
      ["*", "Application/Title", "Capability/Title"],
      ["Application", "Capability"],
      `ApplicationCode eq '${applicationCode}' and IsActive eq true`
    );
    
    // Get all related capabilities
    const capabilityCodes = relationships.map(rel => rel.CapabilityCode);
    
    const capabilities = capabilityCodes.length > 0
      ? await this.getListItems<CapabilityEntity>(
          "Capabilities",
          ["*"],
          [],
          capabilityCodes.map(code => `CapabilityCode eq '${code}'`).join(" or ")
        )
      : [];
    
    // Separate by relationship type
    const primaryRelationships = relationships.filter(rel => rel.RelationshipType === "Primary");
    const supportingRelationships = relationships.filter(rel => rel.RelationshipType === "Supporting");
    
    const primaryCapabilities = capabilities.filter(cap => 
      primaryRelationships.some(rel => rel.CapabilityCode === cap.CapabilityCode)
    );
    
    const supportingCapabilities = capabilities.filter(cap => 
      supportingRelationships.some(rel => rel.CapabilityCode === cap.CapabilityCode)
    );
    
    return {
      application,
      capabilities,
      relationships,
      primaryCapabilities,
      supportingCapabilities
    };
  }
  
  // COMPLEX: Get capability tree with application counts
  async getCapabilityTreeWithApplicationCounts(): Promise<CapabilityTreeNode[]> {
    
    // Get all capabilities
    const allCapabilities = await this.getListItems<CapabilityEntity>(
      "Capabilities",
      ["*"],
      [],
      undefined,
      "Level,CapabilityCode"
    );
    
    // Get all hierarchy relationships
    const hierarchyRelationships = await this.getListItems<CapabilityHierarchyJunction>(
      "CapabilityHierarchy",
      ["*"],
      [],
      "IsActive eq true",
      "SortOrder"
    );
    
    // Get all application-capability relationships
    const appCapRelationships = await this.getListItems<ApplicationCapabilityJunction>(
      "ApplicationCapabilities",
      ["*"],
      [],
      "IsActive eq true"
    );
    
    // Build tree with application counts
    return this.buildCapabilityTree(
      allCapabilities, 
      hierarchyRelationships, 
      appCapRelationships
    );
  }
  
  private buildCapabilityTree(
    capabilities: CapabilityEntity[],
    hierarchyRelationships: CapabilityHierarchyJunction[],
    appCapRelationships: ApplicationCapabilityJunction[]
  ): CapabilityTreeNode[] {
    
    // Create lookup maps
    const capabilityMap = new Map(capabilities.map(cap => [cap.CapabilityCode, cap]));
    const childrenMap = new Map<string, string[]>();
    
    // Build parent-child map
    hierarchyRelationships.forEach(rel => {
      if (!childrenMap.has(rel.ParentCapabilityCode)) {
        childrenMap.set(rel.ParentCapabilityCode, []);
      }
      childrenMap.get(rel.ParentCapabilityCode)!.push(rel.ChildCapabilityCode);
    });
    
    // Count applications per capability
    const appCountMap = new Map<string, number>();
    appCapRelationships.forEach(rel => {
      appCountMap.set(rel.CapabilityCode, (appCountMap.get(rel.CapabilityCode) || 0) + 1);
    });
    
    // Build tree nodes
    const buildNode = (capabilityCode: string): CapabilityTreeNode => {
      const capability = capabilityMap.get(capabilityCode)!;
      const childCodes = childrenMap.get(capabilityCode) || [];
      const children = childCodes.map(buildNode);
      
      const directApplicationCount = appCountMap.get(capabilityCode) || 0;
      const totalApplicationCount = directApplicationCount + 
        children.reduce((sum, child) => sum + child.totalApplicationCount, 0);
      
      return {
        ...capability,
        children,
        directApplicationCount,
        totalApplicationCount,
        hasChildren: children.length > 0
      };
    };
    
    // Find root capabilities (Level 1)
    const rootCapabilities = capabilities.filter(cap => cap.Level === "Level 1");
    
    return rootCapabilities.map(cap => buildNode(cap.CapabilityCode));
  }
}

export type CapabilityTreeNode = CapabilityEntity & {
  children: CapabilityTreeNode[];
  directApplicationCount: number;
  totalApplicationCount: number;
  hasChildren: boolean;
};
```

### Step 3: Create React Components

```typescript
// src/components/CapabilityApplicationsList.tsx
export const CapabilityApplicationsList: React.FC<{
  capabilityCode: string;
  context: WebPartContext;
}> = ({ capabilityCode, context }) => {
  
  const [data, setData] = useState<{
    capability?: CapabilityEntity;
    applications?: ApplicationEntity[];
    relationships?: ApplicationCapabilityJunction[];
  }>({});
  const [loading, setLoading] = useState(true);
  
  const relationshipService = useMemo(() => new BCRRelationshipService(context), [context]);
  
  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        const result = await relationshipService.getApplicationsForCapability(capabilityCode);
        setData(result);
      } catch (error) {
        console.error('Failed to load capability applications:', error);
      } finally {
        setLoading(false);
      }
    };
    
    if (capabilityCode) {
      loadData();
    }
  }, [capabilityCode, relationshipService]);
  
  if (loading) return <div>Loading applications...</div>;
  if (!data.capability) return <div>Capability not found</div>;
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>{data.capability.Title}</CardTitle>
        <CardDescription>
          {data.applications?.length || 0} applications using this capability
        </CardDescription>
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          {data.applications?.map(app => {
            const relationship = data.relationships?.find(rel => rel.ApplicationCode === app.ApplicationCode);
            return (
              <div key={app.Id} className="flex items-center justify-between p-3 border rounded">
                <div>
                  <h4 className="font-medium">{app.Title}</h4>
                  <p className="text-sm text-gray-500">{app.ApplicationCode}</p>
                </div>
                <div className="text-right">
                  <span className={`px-2 py-1 rounded text-xs ${
                    relationship?.RelationshipType === 'Primary' 
                      ? 'bg-blue-100 text-blue-700' 
                      : 'bg-gray-100 text-gray-700'
                  }`}>
                    {relationship?.RelationshipType || 'Unknown'}
                  </span>
                </div>
              </div>
            );
          })}
        </div>
      </CardContent>
    </Card>
  );
};

// src/components/CapabilityHierarchyTree.tsx
export const CapabilityHierarchyTree: React.FC<{
  parentCapabilityCode: string;
  context: WebPartContext;
}> = ({ parentCapabilityCode, context }) => {
  
  const [data, setData] = useState<{
    parentCapability?: CapabilityEntity;
    childCapabilities?: CapabilityEntity[];
  }>({});
  const [loading, setLoading] = useState(true);
  
  const relationshipService = useMemo(() => new BCRRelationshipService(context), [context]);
  
  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        const result = await relationshipService.getSubCapabilities(parentCapabilityCode);
        setData(result);
      } catch (error) {
        console.error('Failed to load sub-capabilities:', error);
      } finally {
        setLoading(false);
      }
    };
    
    if (parentCapabilityCode) {
      loadData();
    }
  }, [parentCapabilityCode, relationshipService]);
  
  if (loading) return <div>Loading sub-capabilities...</div>;
  if (!data.parentCapability) return <div>Parent capability not found</div>;
  
  return (
    <Card>
      <CardHeader>
        <CardTitle>{data.parentCapability.Title}</CardTitle>
        <CardDescription>
          {data.childCapabilities?.length || 0} sub-capabilities
        </CardDescription>
      </CardHeader>
      <CardContent>
        <div className="space-y-2">
          {data.childCapabilities?.map(child => (
            <div key={child.Id} className="flex items-center p-2 border-l-4 border-blue-200">
              <div>
                <h4 className="font-medium">{child.Title}</h4>
                <p className="text-sm text-gray-500">{child.CapabilityCode}</p>
              </div>
            </div>
          ))}
        </div>
      </CardContent>
    </Card>
  );
};

// src/components/ApplicationCapabilitiesView.tsx
export const ApplicationCapabilitiesView: React.FC<{
  applicationCode: string;
  context: WebPartContext;
}> = ({ applicationCode, context }) => {
  
  const [data, setData] = useState<{
    application?: ApplicationEntity;
    primaryCapabilities?: CapabilityEntity[];
    supportingCapabilities?: CapabilityEntity[];
  }>({});
  const [loading, setLoading] = useState(true);
  
  const relationshipService = useMemo(() => new BCRRelationshipService(context), [context]);
  
  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        const result = await relationshipService.getCapabilitiesForApplication(applicationCode);
        setData(result);
      } catch (error) {
        console.error('Failed to load application capabilities:', error);
      } finally {
        setLoading(false);
      }
    };
    
    if (applicationCode) {
      loadData();
    }
  }, [applicationCode, relationshipService]);
  
  if (loading) return <div>Loading capabilities...</div>;
  if (!data.application) return <div>Application not found</div>;
  
  return (
    <div className="space-y-6">
      <Card>
        <CardHeader>
          <CardTitle>{data.application.Title}</CardTitle>
          <CardDescription>{data.application.ApplicationCode}</CardDescription>
        </CardHeader>
      </Card>
      
      {data.primaryCapabilities && data.primaryCapabilities.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle>Primary Capabilities</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
              {data.primaryCapabilities.map(cap => (
                <div key={cap.Id} className="p-3 border rounded bg-blue-50">
                  <h4 className="font-medium">{cap.Title}</h4>
                  <p className="text-sm text-gray-600">{cap.CapabilityCode}</p>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}
      
      {data.supportingCapabilities && data.supportingCapabilities.length > 0 && (
        <Card>
          <CardHeader>
            <CardTitle>Supporting Capabilities</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-3">
              {data.supportingCapabilities.map(cap => (
                <div key={cap.Id} className="p-3 border rounded bg-gray-50">
                  <h4 className="font-medium">{cap.Title}</h4>
                  <p className="text-sm text-gray-600">{cap.CapabilityCode}</p>
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      )}
    </div>
  );
};
```

### Step 4: Main Web Part Component

```typescript
// src/components/BCRExplorer.tsx
export const BCRExplorer: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const [selectedCapability, setSelectedCapability] = useState<string>("");
  const [selectedApplication, setSelectedApplication] = useState<string>("");
  const [viewMode, setViewMode] = useState<'capability' | 'application'>('capability');
  
  return (
    <div className="w-full max-w-6xl mx-auto p-6">
      <Card className="mb-6">
        <CardHeader>
          <CardTitle>BCR Relationship Explorer</CardTitle>
          <CardDescription>
            Explore capabilities, applications, and their relationships
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Tabs value={viewMode} onValueChange={setViewMode}>
            <TabsList>
              <TabsTrigger value="capability">Capability View</TabsTrigger>
              <TabsTrigger value="application">Application View</TabsTrigger>
            </TabsList>
            
            <TabsContent value="capability" className="space-y-6">
              <div>
                <label className="block text-sm font-medium mb-2">
                  Select Capability:
                </label>
                <input
                  type="text"
                  placeholder="Enter capability code (e.g., CAP001)"
                  value={selectedCapability}
                  onChange={(e) => setSelectedCapability(e.target.value)}
                  className="w-full p-2 border rounded"
                />
              </div>
              
              {selectedCapability && (
                <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
                  <CapabilityApplicationsList 
                    capabilityCode={selectedCapability}
                    context={context}
                  />
                  <CapabilityHierarchyTree
                    parentCapabilityCode={selectedCapability}
                    context={context}
                  />
                </div>
              )}
            </TabsContent>
            
            <TabsContent value="application" className="space-y-6">
              <div>
                <label className="block text-sm font-medium mb-2">
                  Select Application:
                </label>
                <input
                  type="text"
                  placeholder="Enter application code (e.g., APP001)"
                  value={selectedApplication}
                  onChange={(e) => setSelectedApplication(e.target.value)}
                  className="w-full p-2 border rounded"
                />
              </div>
              
              {selectedApplication && (
                <ApplicationCapabilitiesView
                  applicationCode={selectedApplication}
                  context={context}
                />
              )}
            </TabsContent>
          </Tabs>
        </CardContent>
      </Card>
    </div>
  );
};
```

## Summary

This complete guide shows you how to:

1. **Excel Power Query**: Generate relationships automatically from business rules
2. **SharePoint Lists**: Import entities and junction tables separately
3. **React Queries**: Query one-to-many and many-to-many relationships efficiently
4. **UI Components**: Display relationship data in a user-friendly interface

The key insight is using **junction tables** to handle relationships cleanly, while **Power Query** automates the tedious relationship generation work that would be manual in other tools.