# Ausgrid BCM Minimal Implementation Guide

Complete implementation using your existing Level ID structure with no artificial business codes.

## SharePoint Lists Structure

### List 1: Capabilities

**List Settings:**
- **List Name**: `Capabilities`
- **Description**: Business capability master list using existing Level ID structure

**Columns:**
```
Column Name      | Type     | Required | Indexed | Choices/Settings
Title           | Text     | Yes      | Yes     | (Default SharePoint field)
LevelID         | Text     | Yes      | Yes     | Unique values like "1.1", "1.2", "1.3"
Tier            | Choice   | Yes      | Yes     | Strategic, Core, Supporting
Level           | Number   | Yes      | Yes     | 0, 1, 2, 3
ParentLevelID   | Text     | No       | Yes     | For hierarchy (e.g., "1.1" is parent of "1.2")
Definition      | Text     | No       | No      | Long description
Owner           | Person   | No       | Yes     | Business owner
```

**Example Data:**
```
Title                              | LevelID | Tier    | Level | ParentLevelID | Definition | Owner
Asset Management                   | 1.1     | Core    | 1     |               | The systematic and coordinated activities... | Murray Chandler
Design & Construction              | 1.2     | Core    | 2     | 1.1           | Planning and executing infrastructure projects... |
Project Design & Planning          | 1.3     | Core    | 3     | 1.2           | Defining a project's goals, structure... |
Construction Management            | 1.3     | Core    | 3     | 1.2           | Overseeing energy infrastructure projects... |
Network Operations                 | 1.1     | Core    | 1     |               | The capability to manage and maintain... |
Network Control & Monitoring       | 1.2     | Core    | 2     | 1.1           | Real-time monitoring and control... |
Strategic Planning Management      | 1.1     | Supporting | 1   |               | Long-term planning processes... |
```

### List 2: Applications

**List Settings:**
- **List Name**: `Applications`
- **Description**: Master application list with standardized names

**Columns:**
```
Column Name        | Type     | Required | Indexed | Settings
Title             | Text     | Yes      | Yes     | Standardized application name
Category          | Choice   | No       | Yes     | ERP, GIS, Analytics, SCADA, Document Management, etc.
Vendor            | Text     | No       | Yes     | SAP, Microsoft, ESRI, Custom, etc.
Description       | Text     | No       | No      | What the application does
Status            | Choice   | No       | Yes     | Active, Deprecated, Planned
```

**Example Data:**
```
Title                    | Category              | Vendor        | Description | Status
SAP PM                  | ERP                   | SAP           | Plant Maintenance Management | Active
Power BI                | Analytics             | Microsoft     | Business Intelligence Platform | Active
MyWorld                 | GIS                   | Custom        | Geographic Information System | Active
GIS Core                | GIS                   | ESRI          | Core GIS Platform | Active
Neara                   | Network Analysis      | Neara         | Network Analytics Platform | Active
Azure IS                | Cloud Platform        | Microsoft     | Azure Integration Services | Active
Cymcap                  | Network Analysis      | CYME          | Cable Capacity Analysis | Active
EDP                     | Data Platform         | Custom        | Enterprise Data Platform | Active
AIMS 3D                 | Asset Management      | Custom        | 3D Asset Information System | Active
Network Viewers         | Visualization         | Custom        | Network Visualization Tools | Active
PMIS                    | Project Management    | Custom        | Project Management Information System | Active
SAP WMS                 | Warehouse Management  | SAP           | Warehouse Management System | Active
RIB CX                  | Construction          | RIB Software  | Construction Management | Active
Autodesk Vault          | Document Management   | Autodesk      | Engineering Document Management | Active
DPRBS                   | Design Management     | Custom        | Design Project Resource Booking System | Active
```

### List 3: CapabilityApplications (Junction Table)

**List Settings:**
- **List Name**: `CapabilityApplications`
- **Description**: Many-to-many relationships between capabilities and applications

**Columns:**
```
Column Name        | Type     | Required | Indexed | Settings
Title             | Text     | Yes      | Yes     | Auto-generated: "LevelID - ApplicationName"
CapabilityLevelID | Text     | Yes      | Yes     | References Capabilities.LevelID
ApplicationName   | Text     | Yes      | Yes     | References Applications.Title
UsageType         | Choice   | No       | Yes     | Primary, Supporting, Optional
Source            | Text     | No       | No      | Where this relationship was identified
```

**Example Data (Your Power BI scenario):**
```
Title                                    | CapabilityLevelID | ApplicationName | UsageType  | Source
1.1 - Power BI                         | 1.1               | Power BI        | Supporting | Asset Management capabilities
1.1 - Power BI                         | 1.1               | Power BI        | Supporting | Network Operations  
2.2 - Power BI                         | 2.2               | Power BI        | Primary    | Customer Analytics
3.1 - Power BI                         | 3.1               | Power BI        | Supporting | Financial Reporting
1.2 - SAP PM                           | 1.2               | SAP PM          | Primary    | Asset Operations Management
1.3 - SAP PM                           | 1.3               | SAP PM          | Primary    | Asset Performance Management
1.2 - MyWorld                          | 1.2               | MyWorld         | Primary    | Digital Asset Intelligence
1.2 - GIS Core                         | 1.2               | GIS Core        | Primary    | Digital Asset Intelligence
1.2 - Neara                            | 1.2               | Neara           | Primary    | Digital Asset Intelligence
```

## Power Query Implementation

### Step 1: Parse and Clean Your CSV Data

```m
let
    // Load the Ausgrid CSV file
    Source = Csv.Document(File.Contents("C:\Path\To\Ausgrid BCM Definition - source-data.csv")),
    Headers = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    
    // Clean the data - remove empty rows and header rows
    CleanData = Table.SelectRows(Headers, each 
        [Tier] <> null and 
        [Tier] <> "" and 
        [Tier] <> "Tier" and
        [#"Level ID"] <> null and
        [#"Level ID"] <> ""
    ),
    
    // Create the Capabilities table
    CapabilitiesTable = Table.SelectColumns(CleanData, {
        "Capability", 
        "Level ID", 
        "Tier", 
        "Level", 
        "Definition",
        "The owner owns the definition & ongoing maintenance"
    }),
    
    // Add parent Level ID based on hierarchy
    AddParentLevelID = Table.AddColumn(CapabilitiesTable, "ParentLevelID", (row) =>
        let
            currentLevelID = row[#"Level ID"],
            level = Number.FromText(Text.From(row[Level]))
        in
            if level <= 1 then null
            else if level = 2 then
                // Level 2 parent is the Level 1 with same first number
                let
                    parts = Text.Split(currentLevelID, "."),
                    parentID = parts{0} & ".1"
                in
                    parentID
            else if level = 3 then
                // Level 3 parent is the Level 2 with same first two numbers
                let
                    parts = Text.Split(currentLevelID, "."),
                    parentID = parts{0} & ".2"
                in
                    parentID
            else null
    ),
    
    // Rename columns for SharePoint
    RenameCapabilityColumns = Table.RenameColumns(AddParentLevelID, {
        {"Capability", "Title"},
        {"Level ID", "LevelID"},
        {"The owner owns the definition & ongoing maintenance", "Owner"}
    })
in
    RenameCapabilityColumns
```

### Step 2: Extract and Standardize Applications

```m
let
    // Load the source data
    Source = Csv.Document(File.Contents("C:\Path\To\Ausgrid BCM Definition - source-data.csv")),
    Headers = Table.PromoteHeaders(Source),
    CleanData = Table.SelectRows(Headers, each [#"Supporting Key Applications"] <> null and [#"Supporting Key Applications"] <> ""),
    
    // Split applications and create individual records
    SplitApplications = Table.ExpandListColumn(
        Table.AddColumn(CleanData, "ApplicationList", (row) =>
            let
                appString = row[#"Supporting Key Applications"],
                splitApps = if appString = null then {} else Text.Split(appString, ",")
            in
                splitApps
        ), "ApplicationList"
    ),
    
    // Clean and standardize application names
    StandardizeApps = Table.AddColumn(SplitApplications, "StandardizedAppName", (row) =>
        let
            rawName = Text.Trim(row[ApplicationList]),
            // Apply standardization rules
            standardName = 
                if Text.Contains(Text.Upper(rawName), "SAP PM") or rawName = "SAP PM" then "SAP PM"
                else if Text.Contains(Text.Upper(rawName), "SAP WMS") then "SAP WMS"
                else if Text.Contains(Text.Upper(rawName), "POWER") and Text.Contains(Text.Upper(rawName), "BI") then "Power BI"
                else if Text.Contains(Text.Upper(rawName), "MYWORLD") or rawName = "My World" then "MyWorld"
                else if Text.Contains(Text.Upper(rawName), "GIS CORE") then "GIS Core"
                else if Text.Contains(Text.Upper(rawName), "NETWORK VIEWER") then "Network Viewers"
                else if Text.Contains(Text.Upper(rawName), "AUTODESK VAULT") then "Autodesk Vault"
                else if Text.Contains(Text.Upper(rawName), "RIB") and Text.Contains(Text.Upper(rawName), "CX") then "RIB CX"
                else if rawName = "AIMS 3D" then "AIMS 3D"
                else if rawName = "PMIS" then "PMIS"
                else if rawName = "EDP" then "EDP"
                else if rawName = "Neara" then "Neara"
                else if rawName = "Cymcap" then "Cymcap"
                else if Text.Contains(rawName, "Azure") and Text.Contains(rawName, "IS") then "Azure IS"
                else rawName
        in
            if standardName = "" then null else standardName
    ),
    
    // Create unique applications list
    FilterValidApps = Table.SelectRows(StandardizeApps, each [StandardizedAppName] <> null),
    UniqueApplications = Table.Distinct(
        Table.SelectColumns(FilterValidApps, {"StandardizedAppName"}),
        {"StandardizedAppName"}
    ),
    
    // Add application metadata
    AddAppMetadata = Table.AddColumn(UniqueApplications, "Category", (row) =>
        let
            appName = row[StandardizedAppName]
        in
            if Text.Contains(Text.Upper(appName), "SAP") then "ERP"
            else if Text.Contains(Text.Upper(appName), "GIS") or appName = "MyWorld" or appName = "Neara" then "GIS"
            else if appName = "Power BI" then "Analytics"
            else if Text.Contains(Text.Upper(appName), "NETWORK") then "Network Analysis"
            else if appName = "Azure IS" then "Cloud Platform"
            else if appName = "Autodesk Vault" then "Document Management"
            else if appName = "PMIS" then "Project Management"
            else if appName = "RIB CX" then "Construction"
            else "Other"
    ),
    
    AddVendor = Table.AddColumn(AddAppMetadata, "Vendor", (row) =>
        let
            appName = row[StandardizedAppName]
        in
            if Text.Contains(Text.Upper(appName), "SAP") then "SAP"
            else if appName = "Power BI" or appName = "Azure IS" then "Microsoft"
            else if Text.Contains(Text.Upper(appName), "GIS") then "ESRI"
            else if appName = "Autodesk Vault" then "Autodesk"
            else if appName = "RIB CX" then "RIB Software"
            else if appName = "Neara" then "Neara"
            else "Custom"
    ),
    
    // Rename for SharePoint
    RenameAppColumns = Table.RenameColumns(AddVendor, {
        {"StandardizedAppName", "Title"}
    }),
    
    // Add status and description
    AddStatus = Table.AddColumn(RenameAppColumns, "Status", each "Active"),
    AddDescription = Table.AddColumn(AddStatus, "Description", (row) =>
        let
            appName = row[Title]
        in
            if appName = "SAP PM" then "Plant Maintenance Management System"
            else if appName = "Power BI" then "Business Intelligence and Analytics Platform"
            else if appName = "MyWorld" then "Geographic Information System"
            else if appName = "GIS Core" then "Core GIS Platform"
            else if appName = "PMIS" then "Project Management Information System"
            else ""
    )
in
    AddDescription
```

### Step 3: Create Capability-Application Junction Table

```m
let
    // Load source data
    Source = Csv.Document(File.Contents("C:\Path\To\Ausgrid BCM Definition - source-data.csv")),
    Headers = Table.PromoteHeaders(Source),
    CleanData = Table.SelectRows(Headers, each 
        [#"Supporting Key Applications"] <> null and 
        [#"Supporting Key Applications"] <> "" and
        [#"Level ID"] <> null
    ),
    
    // Split applications for each capability
    SplitApplications = Table.ExpandListColumn(
        Table.AddColumn(CleanData, "ApplicationList", (row) =>
            Text.Split(row[#"Supporting Key Applications"], ",")
        ), "ApplicationList"
    ),
    
    // Standardize application names (same logic as above)
    StandardizeApps = Table.AddColumn(SplitApplications, "StandardizedAppName", (row) =>
        let
            rawName = Text.Trim(row[ApplicationList]),
            standardName = 
                if Text.Contains(Text.Upper(rawName), "SAP PM") then "SAP PM"
                else if Text.Contains(Text.Upper(rawName), "POWER") and Text.Contains(Text.Upper(rawName), "BI") then "Power BI"
                else if Text.Contains(Text.Upper(rawName), "MYWORLD") then "MyWorld"
                else if Text.Contains(Text.Upper(rawName), "GIS CORE") then "GIS Core"
                else if rawName = "Neara" then "Neara"
                else if rawName = "EDP" then "EDP"
                else if rawName = "PMIS" then "PMIS"
                else rawName
        in
            if standardName = "" then null else standardName
    ),
    
    // Create junction table
    CreateJunction = Table.SelectColumns(StandardizeApps, {
        "Level ID", 
        "Capability",
        "StandardizedAppName",
        "Tier"
    }),
    
    // Filter out null applications
    FilterValidJunctions = Table.SelectRows(CreateJunction, each [StandardizedAppName] <> null),
    
    // Add usage type based on tier and context
    AddUsageType = Table.AddColumn(FilterValidJunctions, "UsageType", (row) =>
        let
            tier = row[Tier],
            appName = row[StandardizedAppName]
        in
            if tier = "Core" then "Primary"
            else if tier = "Supporting" then "Supporting"
            else if appName = "Power BI" then "Supporting"  // Power BI is usually supporting
            else "Primary"
    ),
    
    // Create title and rename columns
    AddTitle = Table.AddColumn(AddUsageType, "Title", (row) =>
        row[#"Level ID"] & " - " & row[StandardizedAppName]
    ),
    
    RenameJunctionColumns = Table.RenameColumns(AddTitle, {
        {"Level ID", "CapabilityLevelID"},
        {"StandardizedAppName", "ApplicationName"},
        {"Capability", "Source"}
    }),
    
    // Select final columns
    FinalJunctionTable = Table.SelectColumns(RenameJunctionColumns, {
        "Title",
        "CapabilityLevelID", 
        "ApplicationName",
        "UsageType",
        "Source"
    })
in
    FinalJunctionTable
```

## React Query Service

```typescript
// src/lib/ausgrid-bcm-service.ts
export class AusgridBCMService extends SharePointService {
  
  // Get all applications for a capability (your Power BI example)
  async getApplicationsForCapability(levelID: string): Promise<{
    capability: AusgridCapability;
    applications: AusgridApplication[];
    relationships: CapabilityApplicationJunction[];
  }> {
    
    // Get the capability
    const capabilities = await this.getListItems<AusgridCapability>(
      "Capabilities",
      ["*", "Owner/Title", "Owner/Email"],
      ["Owner"],
      `LevelID eq '${levelID}'`
    );
    
    if (capabilities.length === 0) {
      throw new Error(`Capability with Level ID ${levelID} not found`);
    }
    
    const capability = capabilities[0];
    
    // Get all junction records for this capability
    const relationships = await this.getListItems<CapabilityApplicationJunction>(
      "CapabilityApplications",
      ["*"],
      [],
      `CapabilityLevelID eq '${levelID}'`
    );
    
    // Get unique application names
    const applicationNames = [...new Set(relationships.map(rel => rel.ApplicationName))];
    
    // Get application details
    const applications = applicationNames.length > 0 
      ? await this.getListItems<AusgridApplication>(
          "Applications",
          ["*"],
          [],
          applicationNames.map(name => `Title eq '${name}'`).join(" or ")
        )
      : [];
    
    return {
      capability,
      applications,
      relationships
    };
  }
  
  // Get all capabilities using an application (your Power BI query)
  async getCapabilitiesForApplication(applicationName: string): Promise<{
    application: AusgridApplication;
    capabilities: AusgridCapability[];
    relationships: CapabilityApplicationJunction[];
    primaryCapabilities: AusgridCapability[];
    supportingCapabilities: AusgridCapability[];
  }> {
    
    // Get the application
    const applications = await this.getListItems<AusgridApplication>(
      "Applications",
      ["*"],
      [],
      `Title eq '${applicationName}'`
    );
    
    if (applications.length === 0) {
      throw new Error(`Application ${applicationName} not found`);
    }
    
    const application = applications[0];
    
    // Get all junction records for this application
    const relationships = await this.getListItems<CapabilityApplicationJunction>(
      "CapabilityApplications",
      ["*"],
      [],
      `ApplicationName eq '${applicationName}'`
    );
    
    // Get unique capability Level IDs
    const levelIDs = [...new Set(relationships.map(rel => rel.CapabilityLevelID))];
    
    // Get capability details
    const capabilities = levelIDs.length > 0
      ? await this.getListItems<AusgridCapability>(
          "Capabilities",
          ["*", "Owner/Title", "Owner/Email"],
          ["Owner"],
          levelIDs.map(id => `LevelID eq '${id}'`).join(" or ")
        )
      : [];
    
    // Separate by usage type
    const primaryRelationships = relationships.filter(rel => rel.UsageType === "Primary");
    const supportingRelationships = relationships.filter(rel => rel.UsageType === "Supporting");
    
    const primaryCapabilities = capabilities.filter(cap => 
      primaryRelationships.some(rel => rel.CapabilityLevelID === cap.LevelID)
    );
    
    const supportingCapabilities = capabilities.filter(cap => 
      supportingRelationships.some(rel => rel.CapabilityLevelID === cap.LevelID)
    );
    
    return {
      application,
      capabilities,
      relationships,
      primaryCapabilities,
      supportingCapabilities
    };
  }
  
  // Get capability hierarchy tree
  async getCapabilityHierarchy(): Promise<AusgridCapabilityTreeNode[]> {
    
    // Get all capabilities
    const allCapabilities = await this.getListItems<AusgridCapability>(
      "Capabilities",
      ["*", "Owner/Title", "Owner/Email"],
      ["Owner"],
      undefined,
      "Level,LevelID"
    );
    
    // Get application counts for each capability
    const allRelationships = await this.getListItems<CapabilityApplicationJunction>(
      "CapabilityApplications",
      ["CapabilityLevelID", "ApplicationName"],
      []
    );
    
    // Build application count map
    const appCountMap = new Map<string, number>();
    allRelationships.forEach(rel => {
      const current = appCountMap.get(rel.CapabilityLevelID) || 0;
      appCountMap.set(rel.CapabilityLevelID, current + 1);
    });
    
    return this.buildCapabilityTree(allCapabilities, appCountMap);
  }
  
  private buildCapabilityTree(
    capabilities: AusgridCapability[], 
    appCountMap: Map<string, number>
  ): AusgridCapabilityTreeNode[] {
    
    // Create lookup maps
    const capabilityMap = new Map(capabilities.map(cap => [cap.LevelID, cap]));
    const childrenMap = new Map<string, string[]>();
    
    // Build parent-child relationships
    capabilities.forEach(cap => {
      if (cap.ParentLevelID) {
        if (!childrenMap.has(cap.ParentLevelID)) {
          childrenMap.set(cap.ParentLevelID, []);
        }
        childrenMap.get(cap.ParentLevelID)!.push(cap.LevelID);
      }
    });
    
    // Build tree nodes recursively
    const buildNode = (levelID: string): AusgridCapabilityTreeNode => {
      const capability = capabilityMap.get(levelID)!;
      const childLevelIDs = childrenMap.get(levelID) || [];
      const children = childLevelIDs.map(buildNode);
      
      const directApplicationCount = appCountMap.get(levelID) || 0;
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
    
    // Find root capabilities (Level 1 or those without parents)
    const rootCapabilities = capabilities.filter(cap => 
      cap.Level === 1 || !cap.ParentLevelID
    );
    
    return rootCapabilities.map(cap => buildNode(cap.LevelID));
  }
}

// Type definitions
export type AusgridCapability = SharePointBaseItem & {
  LevelID: string;           // "1.1", "1.2", "1.3"
  Tier: "Strategic" | "Core" | "Supporting";
  Level: number;             // 0, 1, 2, 3
  ParentLevelID?: string;    // Parent Level ID
  Definition: string;
  Owner?: SPUser;
};

export type AusgridApplication = SharePointBaseItem & {
  Category?: string;         // "ERP", "GIS", "Analytics"
  Vendor?: string;           // "SAP", "Microsoft", "Custom"
  Description?: string;
  Status: "Active" | "Deprecated" | "Planned";
};

export type CapabilityApplicationJunction = SharePointBaseItem & {
  CapabilityLevelID: string; // "1.1"
  ApplicationName: string;   // "Power BI"
  UsageType: "Primary" | "Supporting" | "Optional";
  Source?: string;           // Where relationship was identified
};

export type AusgridCapabilityTreeNode = AusgridCapability & {
  children: AusgridCapabilityTreeNode[];
  directApplicationCount: number;
  totalApplicationCount: number;
  hasChildren: boolean;
};
```

## React Components

```typescript
// src/components/PowerBIUsageExample.tsx
export const PowerBIUsageExample: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const [powerBIData, setPowerBIData] = useState<{
    application?: AusgridApplication;
    primaryCapabilities?: AusgridCapability[];
    supportingCapabilities?: AusgridCapability[];
  }>({});
  const [loading, setLoading] = useState(true);
  
  const bcmService = useMemo(() => new AusgridBCMService(context), [context]);
  
  useEffect(() => {
    const loadPowerBIUsage = async () => {
      try {
        setLoading(true);
        const result = await bcmService.getCapabilitiesForApplication("Power BI");
        setPowerBIData(result);
      } catch (error) {
        console.error('Failed to load Power BI usage:', error);
      } finally {
        setLoading(false);
      }
    };
    
    loadPowerBIUsage();
  }, [bcmService]);
  
  if (loading) return <div>Loading Power BI usage...</div>;
  
  return (
    <div className="space-y-6">
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center gap-2">
            <span>📊</span>
            Power BI Usage Analysis
          </CardTitle>
          <CardDescription>
            Showing all capabilities that use Power BI across the organization
          </CardDescription>
        </CardHeader>
      </Card>
      
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        {/* Primary Usage */}
        {powerBIData.primaryCapabilities && powerBIData.primaryCapabilities.length > 0 && (
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Primary Usage</CardTitle>
              <CardDescription>
                Capabilities where Power BI is a primary tool
              </CardDescription>
            </CardHeader>
            <CardContent>
              <div className="space-y-3">
                {powerBIData.primaryCapabilities.map(cap => (
                  <div key={cap.Id} className="p-3 border-l-4 border-blue-500 bg-blue-50">
                    <div className="flex justify-between items-start">
                      <div>
                        <h4 className="font-medium">{cap.Title}</h4>
                        <p className="text-sm text-gray-600">{cap.LevelID}</p>
                        <p className="text-sm text-gray-500 mt-1">{cap.Tier}</p>
                      </div>
                      <span className="text-xs bg-blue-100 text-blue-700 px-2 py-1 rounded">
                        Primary
                      </span>
                    </div>
                  </div>
                ))}
              </div>
            </CardContent>
          </Card>
        )}
        
        {/* Supporting Usage */}
        {powerBIData.supportingCapabilities && powerBIData.supportingCapabilities.length > 0 && (
          <Card>
            <CardHeader>
              <CardTitle className="text-lg">Supporting Usage</CardTitle>
              <CardDescription>
                Capabilities where Power BI provides supporting functionality
              </CardDescription>
            </CardHeader>
            <CardContent>
              <div className="space-y-3">
                {powerBIData.supportingCapabilities.map(cap => (
                  <div key={cap.Id} className="p-3 border-l-4 border-gray-300 bg-gray-50">
                    <div className="flex justify-between items-start">
                      <div>
                        <h4 className="font-medium">{cap.Title}</h4>
                        <p className="text-sm text-gray-600">{cap.LevelID}</p>
                        <p className="text-sm text-gray-500 mt-1">{cap.Tier}</p>
                      </div>
                      <span className="text-xs bg-gray-100 text-gray-700 px-2 py-1 rounded">
                        Supporting
                      </span>
                    </div>
                  </div>
                ))}
              </div>
            </CardContent>
          </Card>
        )}
      </div>
    </div>
  );
};

// src/components/CapabilityApplicationsView.tsx
export const CapabilityApplicationsView: React.FC<{
  levelID: string;
  context: WebPartContext;
}> = ({ levelID, context }) => {
  const [data, setData] = useState<{
    capability?: AusgridCapability;
    applications?: AusgridApplication[];
    relationships?: CapabilityApplicationJunction[];
  }>({});
  const [loading, setLoading] = useState(true);
  
  const bcmService = useMemo(() => new AusgridBCMService(context), [context]);
  
  useEffect(() => {
    const loadData = async () => {
      try {
        setLoading(true);
        const result = await bcmService.getApplicationsForCapability(levelID);
        setData(result);
      } catch (error) {
        console.error('Failed to load capability applications:', error);
      } finally {
        setLoading(false);
      }
    };
    
    if (levelID) {
      loadData();
    }
  }, [levelID, bcmService]);
  
  if (loading) return <div>Loading applications...</div>;
  if (!data.capability) return <div>Capability not found</div>;
  
  return (
    <div className="space-y-4">
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center justify-between">
            <div>
              <h3>{data.capability.Title}</h3>
              <p className="text-sm text-gray-500 font-normal">
                {data.capability.LevelID} • {data.capability.Tier}
              </p>
            </div>
            <span className="text-sm bg-blue-100 text-blue-700 px-3 py-1 rounded-full">
              {data.applications?.length || 0} Applications
            </span>
          </CardTitle>
          {data.capability.Definition && (
            <CardDescription>{data.capability.Definition}</CardDescription>
          )}
        </CardHeader>
      </Card>
      
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {data.applications?.map(app => {
          const relationship = data.relationships?.find(rel => rel.ApplicationName === app.Title);
          return (
            <Card key={app.Id} className="hover:shadow-md transition-shadow">
              <CardContent className="p-4">
                <div className="flex justify-between items-start mb-2">
                  <h4 className="font-medium">{app.Title}</h4>
                  <span className={`text-xs px-2 py-1 rounded ${
                    relationship?.UsageType === 'Primary' 
                      ? 'bg-green-100 text-green-700' 
                      : 'bg-gray-100 text-gray-700'
                  }`}>
                    {relationship?.UsageType || 'Unknown'}
                  </span>
                </div>
                {app.Category && (
                  <p className="text-sm text-gray-600 mb-1">{app.Category}</p>
                )}
                {app.Vendor && (
                  <p className="text-sm text-gray-500">{app.Vendor}</p>
                )}
                {app.Description && (
                  <p className="text-xs text-gray-400 mt-2">{app.Description}</p>
                )}
              </CardContent>
            </Card>
          );
        })}
      </div>
    </div>
  );
};

// src/components/AusgridBCMExplorer.tsx  
export const AusgridBCMExplorer: React.FC<{ context: WebPartContext }> = ({ context }) => {
  const [selectedLevelID, setSelectedLevelID] = useState<string>("1.1");
  const [viewMode, setViewMode] = useState<'capability' | 'application'>('capability');
  
  return (
    <div className="w-full max-w-7xl mx-auto p-6">
      <Card className="mb-6">
        <CardHeader>
          <CardTitle>Ausgrid Business Capability Model Explorer</CardTitle>
          <CardDescription>
            Explore capabilities and applications using your existing Level ID structure
          </CardDescription>
        </CardHeader>
        <CardContent>
          <Tabs value={viewMode} onValueChange={setViewMode}>
            <TabsList>
              <TabsTrigger value="capability">By Capability</TabsTrigger>
              <TabsTrigger value="application">By Application</TabsTrigger>
            </TabsList>
            
            <TabsContent value="capability" className="space-y-6">
              <div>
                <label className="block text-sm font-medium mb-2">
                  Select Capability Level ID:
                </label>
                <select
                  value={selectedLevelID}
                  onChange={(e) => setSelectedLevelID(e.target.value)}
                  className="w-full p-2 border rounded"
                >
                  <option value="1.1">1.1 - Asset Management</option>
                  <option value="1.2">1.2 - Design & Construction</option>
                  <option value="1.3">1.3 - Project Design & Planning</option>
                  <option value="2.1">2.1 - Network Operations</option>
                  <option value="2.2">2.2 - Network Control & Monitoring</option>
                  {/* Add more options based on your data */}
                </select>
              </div>
              
              {selectedLevelID && (
                <CapabilityApplicationsView 
                  levelID={selectedLevelID}
                  context={context}
                />
              )}
            </TabsContent>
            
            <TabsContent value="application" className="space-y-6">
              <PowerBIUsageExample context={context} />
            </TabsContent>
          </Tabs>
        </CardContent>
      </Card>
    </div>
  );
};
```

## Summary

**Your junction table approach is exactly right!** The `CapabilityApplications` list will have duplicate application names:

```
ApplicationName | CapabilityLevelID | UsageType
Power BI       | 1.1               | Supporting
Power BI       | 1.2               | Supporting  
Power BI       | 2.1               | Primary
Power BI       | 3.2               | Supporting
SAP PM         | 1.2               | Primary
SAP PM         | 1.3               | Primary
```

And when you query "all capabilities using Power BI", you simply filter the junction table for `ApplicationName eq 'Power BI'` and get all the related capabilities.

**Key Benefits of This Approach:**
✅ **No artificial codes** - uses your existing Level ID structure
✅ **Handles many-to-many** - Power BI can be used by multiple capabilities
✅ **Minimal complexity** - three simple lists
✅ **Scalable** - easy to add new applications and relationships
✅ **Uses your real data** - no theoretical examples

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Write comprehensive SharePoint Lists relational database guidance", "status": "completed", "priority": "high"}, {"id": "2", "content": "Review IA image when provided by user", "status": "completed", "priority": "high"}, {"id": "3", "content": "Create guidance for importing spreadsheets to SharePoint Lists", "status": "completed", "priority": "high"}, {"id": "4", "content": "Update existing SharePoint lists documentation with relational patterns", "status": "completed", "priority": "medium"}, {"id": "5", "content": "Create BCR-specific SharePoint list schema based on IA diagram", "status": "completed", "priority": "high"}, {"id": "6", "content": "Design hierarchical capability structure for SharePoint Lists", "status": "completed", "priority": "high"}, {"id": "7", "content": "Document clean junction table approach for relational imports", "status": "completed", "priority": "high"}, {"id": "8", "content": "Create complete Ausgrid minimal approach implementation", "status": "completed", "priority": "high"}, {"id": "9", "content": "Build Power Query solution for Ausgrid data", "status": "completed", "priority": "high"}, {"id": "10", "content": "Create React components for Ausgrid BCM system", "status": "completed", "priority": "high"}]