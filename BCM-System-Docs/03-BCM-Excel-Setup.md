# Excel Data Setup Guide
**📖 Page 3 of 5 | Data Setup**

Step-by-step guide to prepare your Excel workbook for the Business Capability Model system.

---
**Navigation:** [📋 Table of Contents](./00-Table-of-Contents.md) | [◀️ Previous: Quick Start](./02-BCM-Getting-Started.md) | **Page 3** | [▶️ Next: Development Guide](./04-BCM-Web-Part-Guide.md)  
---

## Overview

You'll create one Excel workbook with 3 sheets that store all your BCM data. Business users will edit this Excel file when they need to update capabilities or application relationships.

## Excel Workbook Structure

### Sheet 1: Capabilities
**Purpose**: List all your business capabilities with simple hierarchy

**Columns**:
- **A: CapabilityName** - The capability name (e.g., "Asset Management")
- **B: ParentCapabilityName** - Parent capability if this is a sub-capability  
- **C: Level** - Number: 1, 2, or 3 (for display purposes)
- **D: Tier** - Text: "Strategic", "Core", or "Supporting"
- **E: Definition** - Description of what this capability does
- **F: Owner** - Person responsible for this capability

**Example**:
```
CapabilityName          | ParentCapabilityName | Level | Tier    | Definition                    | Owner
Asset Management        |                      | 1     | Core    | Managing physical assets...   | John Smith
Design & Construction   | Asset Management     | 2     | Core    | Planning infrastructure...    | Jane Doe
Network Operations      |                      | 1     | Core    | Operating the network...      | Bob Wilson
```

### Sheet 2: Applications  
**Purpose**: List all your software applications

**Columns**:
- **A: ApplicationName** - The application name (e.g., "Power BI") 
- **B: Category** - Type of application (e.g., "Analytics", "ERP")
- **C: Vendor** - Who makes it (e.g., "Microsoft", "SAP") 
- **D: Status** - "Active", "Deprecated", or "Planned"

**Example**:
```
ApplicationName | Category  | Vendor    | Status
Power BI       | Analytics | Microsoft | Active
SAP PM         | ERP       | SAP       | Active  
MyWorld        | GIS       | Custom    | Active
Excel          | Analytics | Microsoft | Active
```

### Sheet 3: CapabilityApplications
**Purpose**: The relationships - which applications support which capabilities

**Columns**:
- **A: CapabilityName** - Must match a name from the Capabilities sheet
- **B: ApplicationName** - Must match a name from the Applications sheet
- **C: UsageType** - "Primary" or "Supporting" 
- **D: Notes** - Optional comments about this relationship

**Example**:
```
CapabilityName      | ApplicationName | UsageType  | Notes
Asset Management    | Power BI        | Supporting | Dashboards and reporting
Asset Management    | SAP PM          | Primary    | Core maintenance system
Network Operations  | Power BI        | Primary    | Operations dashboards
Network Operations  | Excel           | Supporting | Ad-hoc analysis
Design & Construction| MyWorld        | Primary    | GIS mapping
```

## Step-by-Step Setup

### Step 1: Create the Excel Workbook

1. **Open Excel Online** (or Excel desktop)
2. **Create a new blank workbook**
3. **Save it as**: `BCM-Data.xlsx`
4. **Rename the first sheet** to `Capabilities`
5. **Add two more sheets**: `Applications` and `CapabilityApplications`

### Step 2: Set Up the Capabilities Sheet

1. **Go to the Capabilities sheet**
2. **Add headers in row 1**:
   - A1: `CapabilityName`
   - B1: `ParentCapabilityName` 
   - C1: `Level`
   - D1: `Tier`
   - E1: `Definition`
   - F1: `Owner`

3. **Add your capability data** starting in row 2
4. **Keep it simple**: Use clear, consistent names
5. **For hierarchy**: Leave ParentCapabilityName blank for top-level capabilities

### Step 3: Set Up the Applications Sheet

1. **Go to the Applications sheet**
2. **Add headers in row 1**:
   - A1: `ApplicationName`
   - B1: `Category`
   - C1: `Vendor`
   - D1: `Status`

3. **Add your applications** starting in row 2
4. **Use consistent names**: "Power BI" not "PowerBI" or "Power-BI"

### Step 4: Set Up the Relationships Sheet

1. **Go to the CapabilityApplications sheet**
2. **Add headers in row 1**:
   - A1: `CapabilityName`
   - B1: `ApplicationName`
   - C1: `UsageType`
   - D1: `Notes`

3. **Add relationships** starting in row 2
4. **Use exact names**: Must match exactly what's in the other sheets
5. **Bulk editing tip**: Copy capability names down for multiple apps

### Step 5: Data Validation (Optional but Recommended)

**For the CapabilityApplications sheet**:

1. **Select column A** (CapabilityName)
2. **Data → Data Validation**
3. **Allow**: List
4. **Source**: `=Capabilities!A:A`
5. **Repeat for column B** using `=Applications!A:A`

This prevents typos and ensures relationships are valid.

### Step 6: Upload to SharePoint

1. **Save your Excel file**
2. **Go to your SharePoint site**
3. **Upload to a Document Library** (or save to OneDrive for Business)
4. **Make sure your web part developers can access it**

## Data Entry Tips

### For Business Users

**Adding new capabilities**:
- Add to the Capabilities sheet first
- Then add relationships in CapabilityApplications sheet

**Adding application relationships**:
- Use copy/paste to quickly add multiple relationships
- Filter or sort to find existing data
- Use "Fill Down" for repetitive data entry

**Bulk updates**:
- Excel's filter and sort features work great
- Copy/paste from other spreadsheets
- Use Find & Replace for bulk changes

### Data Quality Tips

✅ **Be consistent** with naming (case-sensitive)  
✅ **Use clear, full names** ("Customer Management" not "CustMgt")  
✅ **Keep definitions brief** but descriptive  
✅ **Update regularly** rather than big batches  

❌ **Don't use special characters** in names  
❌ **Don't leave gaps** in your data  
❌ **Don't use abbreviations** unless necessary  

## What Happens Next

Once your Excel workbook is ready:
1. **Your developers** will connect it to the SharePoint web part
2. **End users** will explore your data through the web interface  
3. **You can continue editing** the Excel file anytime
4. **Changes appear automatically** in the web part

The Excel file becomes your simple, maintenance-free database that anyone can understand and edit.

## Adding More Data Types

Want to track additional relationships? Here's how to extend your Excel workbook with more entity types.

### Example 1: Adding Application Integrations

**New Sheet: ApplicationIntegrations**
Track which applications integrate with each other.

**Columns**:
- **A: SourceApplication** - The application sending data
- **B: TargetApplication** - The application receiving data  
- **C: IntegrationType** - "API", "File Transfer", "Database", "Real-time"
- **D: DataFlowDirection** - "One-way", "Two-way"
- **E: Criticality** - "Low", "Medium", "High", "Critical"
- **F: Notes** - Integration details

**Example Data**:
```
SourceApplication | TargetApplication | IntegrationType | DataFlowDirection | Criticality | Notes
SAP PM           | Power BI          | API             | One-way          | High        | Maintenance data for dashboards
CRM System       | Power BI          | Database        | One-way          | Medium      | Customer analytics
MyWorld          | SAP PM            | File Transfer   | Two-way          | High        | Asset location updates
Excel            | Power BI          | File Transfer   | One-way          | Low         | Ad-hoc data imports
```

**Usage**: "Which applications integrate with Power BI?" or "What's the data flow from SAP PM?"

### Example 2: Adding Business Units

**New Sheet: BusinessUnits**
Track organizational structure.

**Columns**:
- **A: BusinessUnitName** - The business unit name
- **B: ParentBusinessUnit** - Parent unit (for hierarchy)
- **C: BusinessUnitHead** - Person in charge
- **D: Location** - Physical or organizational location
- **E: CostCenter** - Financial tracking code

**Example Data**:
```
BusinessUnitName        | ParentBusinessUnit | BusinessUnitHead | Location  | CostCenter
Operations             |                    | Sarah Johnson    | Sydney    | CC001
Network Operations     | Operations         | Mike Chen        | Sydney    | CC001-01
Customer Services      | Operations         | Lisa Wong        | Melbourne | CC001-02
Engineering           |                    | David Smith      | Brisbane  | CC002
Design Engineering    | Engineering        | Emma Brown       | Brisbane  | CC002-01
```

**New Relationship Sheet: CapabilityBusinessUnits**
Link capabilities to business units.

**Columns**:
- **A: CapabilityName** - Must match Capabilities sheet
- **B: BusinessUnitName** - Must match BusinessUnits sheet
- **C: Responsibility** - "Owner", "Stakeholder", "User"
- **D: Notes** - Additional context

**Example Data**:
```
CapabilityName      | BusinessUnitName    | Responsibility | Notes
Asset Management    | Operations          | Owner          | Primary responsibility
Asset Management    | Engineering         | Stakeholder    | Technical input
Network Operations  | Network Operations  | Owner          | Day-to-day operations
Customer Management | Customer Services   | Owner          | Direct customer contact
```

### Example 3: Multiple Relationship Types

You can have as many relationship sheets as needed:

**Current sheets**:
- CapabilityApplications (capabilities ↔ applications)

**Additional relationship sheets**:
- ApplicationIntegrations (applications ↔ applications)
- CapabilityBusinessUnits (capabilities ↔ business units)
- ApplicationBusinessUnits (applications ↔ business units)
- BusinessUnitIntegrations (business units ↔ business units)

### Setting Up Additional Sheets

**Step 1: Plan Your Data**
- What entities do you want to track?
- What relationships matter to your organization?
- Keep it simple - start with what you'll actually use

**Step 2: Add New Sheets**
- Right-click sheet tabs → Insert
- Name them clearly (e.g., "BusinessUnits", "ApplicationIntegrations")
- Follow the same column naming pattern

**Step 3: Set Up Data Validation**
- Use dropdown lists referencing your master sheets
- Example: `=BusinessUnits!A:A` for business unit names
- Prevents typos and ensures data consistency

**Step 4: Update Your Web Part**
Add new query methods to read the additional sheets:

```typescript
// Example: Get integrations for an application
async getIntegrationsForApplication(applicationName: string) {
  const response = await this.graphClient.get(
    `/workbook/worksheets/ApplicationIntegrations/usedRange`
  );
  
  return response.values
    .slice(1)
    .filter(row => row[0] === applicationName || row[1] === applicationName)
    .map(row => ({
      sourceApp: row[0],
      targetApp: row[1], 
      integrationType: row[2],
      dataFlow: row[3],
      criticality: row[4],
      notes: row[5]
    }));
}
```

### Real-World Example: Complete Ausgrid Setup

Based on your CSV data, you might want:

**Core Sheets**:
- Capabilities (your existing capability hierarchy)
- Applications (Power BI, SAP PM, MyWorld, etc.)
- BusinessUnits (Operations, Engineering, Customer Services)

**Relationship Sheets**:
- CapabilityApplications (which apps support which capabilities)
- CapabilityBusinessUnits (which business units own which capabilities)
- ApplicationIntegrations (how applications connect to each other)

**Example Query**: "Show me all applications used by the Operations business unit"
1. Look up capabilities owned by Operations in CapabilityBusinessUnits
2. Find applications for those capabilities in CapabilityApplications  
3. Display the complete relationship chain

### Tips for Complex Data

**Keep relationships simple**:
- One relationship per row in relationship sheets
- Use consistent naming across all sheets
- Don't try to put multiple relationships in one cell

**Start small and grow**:
- Begin with just capabilities and applications
- Add business units when you need them
- Add integrations when they become important

**Use Excel's power**:
- Pivot tables to analyze your relationship data
- Conditional formatting to highlight important relationships
- Filters to focus on specific business units or application types

The beauty of this approach is that you can add as much complexity as you need while keeping the core system simple and maintainable.