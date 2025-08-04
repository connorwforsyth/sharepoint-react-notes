# Web Part Development Guide
**📖 Page 4 of 5 | Development Guide**

Simple guide to building the SharePoint web part that reads from your Excel Online data.

---
**Navigation:** [📋 Table of Contents](./00-Table-of-Contents.md) | [◀️ Previous: Data Setup](./03-BCM-Excel-Setup.md) | **Page 4** | [▶️ Next: Quick Reference](./05-BCM-Quick-Reference.md)  
---

## What You're Building

A SharePoint web part that:
- Reads data from your Excel workbook automatically
- Shows which applications support each capability  
- Lets users explore capability hierarchies
- Updates in real-time when Excel data changes

No complex database setup, no maintenance - just Excel and React.

## Prerequisites  

✅ **SharePoint Framework development environment** set up  
✅ **Excel workbook** created and uploaded ([Excel Setup Guide](./03-BCM-Excel-Setup.md))  
✅ **Basic React knowledge**  

## Project Setup

### Create SharePoint Framework Project

```bash
# Create new SPFx project
yo @microsoft/sharepoint

# Choose:
# - React framework
# - TypeScript
# - Your solution name (e.g., "bcm-explorer")
```

### Add Required Dependencies

```bash
# Add UI components (optional but recommended)
npm install @fluentui/react lucide-react

# Microsoft Graph is already included in SPFx
```

## TypeScript Type Definitions

### Step 1: Define Your Data Types

Create `src/types/bcm-types.ts` for type safety:

```typescript
// Core entity types matching your Excel sheets
export type Capability = {
  capabilityName: string;
  parentCapabilityName?: string;
  level: 1 | 2 | 3;
  tier: 'Strategic' | 'Core' | 'Supporting';
  definition?: string;
  owner?: string;
};

export type Application = {
  applicationName: string;
  category?: string;
  vendor?: string;
  status: 'Active' | 'Deprecated' | 'Planned';
};

export type CapabilityApplication = {
  capabilityName: string;
  applicationName: string;
  usageType: 'Primary' | 'Supporting';
  notes?: string;
};

// Extended types for additional data (if you add them)
export type BusinessUnit = {
  businessUnitName: string;
  parentBusinessUnit?: string;
  businessUnitHead?: string;
  location?: string;
  costCenter?: string;
};

export type ApplicationIntegration = {
  sourceApplication: string;
  targetApplication: string;
  integrationType: 'API' | 'File Transfer' | 'Database' | 'Real-time';
  dataFlowDirection: 'One-way' | 'Two-way';
  criticality: 'Low' | 'Medium' | 'High' | 'Critical';
  notes?: string;
};

export type CapabilityBusinessUnit = {
  capabilityName: string;
  businessUnitName: string;
  responsibility: 'Owner' | 'Stakeholder' | 'User';
  notes?: string;
};
```

**Why types are perfect for this system:**
✅ **Exact Excel column mapping** - each type matches your sheet structure  
✅ **Union types** - precise values like `'Primary' | 'Supporting'`  
✅ **Optional fields** - marked with `?` for flexible data entry  
✅ **Type safety** - catches errors at compile time  
✅ **No maintenance** - update once when Excel structure changes  

## Core Implementation

### Step 2: Excel Service

Create `src/services/ExcelBCMService.ts`:

```typescript
import { MSGraphClientV3, MSGraphClientFactory } from '@microsoft/sp-http';
import { WebPartContext } from '@microsoft/sp-webpart-base';
import { CapabilityApplication, Application, Capability } from '../types/bcm-types';

export class ExcelBCMService {
  private graphClient: MSGraphClientV3;
  private workbookId: string;

  constructor(context: WebPartContext, workbookId: string) {
    this.workbookId = workbookId;
    
    // Get Microsoft Graph client (no setup needed)
    context.serviceScope.consume(MSGraphClientFactory.serviceKey)
      .getClient("3")
      .then(client => this.graphClient = client);
  }

  // Get all applications for a specific capability
  async getApplicationsForCapability(capabilityName: string): Promise<CapabilityApplication[]> {
    const response = await this.graphClient.get(
      `/me/drive/items/${this.workbookId}/workbook/worksheets/CapabilityApplications/usedRange`
    );
    
    // Filter and map with proper typing
    return response.values
      .slice(1) // Skip header row
      .filter((row: any[]) => row[0] === capabilityName)
      .map((row: any[]): CapabilityApplication => ({
        capabilityName: row[0],
        applicationName: row[1], 
        usageType: row[2] as 'Primary' | 'Supporting',
        notes: row[3]
      }));
  }

  // Get all capabilities that use a specific application  
  async getCapabilitiesForApplication(applicationName: string): Promise<CapabilityApplication[]> {
    const response = await this.graphClient.get(
      `/me/drive/items/${this.workbookId}/workbook/worksheets/CapabilityApplications/usedRange`
    );
    
    // Filter and map with proper typing
    return response.values
      .slice(1)
      .filter((row: any[]) => row[1] === applicationName)
      .map((row: any[]): CapabilityApplication => ({
        capabilityName: row[0],
        applicationName: row[1],
        usageType: row[2] as 'Primary' | 'Supporting',
        notes: row[3]
      }));
  }

  // Get all capabilities with proper typing
  async getAllCapabilities(): Promise<Capability[]> {
    const response = await this.graphClient.get(
      `/me/drive/items/${this.workbookId}/workbook/worksheets/Capabilities/usedRange`
    );
    
    return response.values
      .slice(1)
      .map((row: any[]): Capability => ({
        capabilityName: row[0],
        parentCapabilityName: row[1] || undefined,
        level: row[2] as 1 | 2 | 3,
        tier: row[3] as 'Strategic' | 'Core' | 'Supporting',
        definition: row[4] || undefined,
        owner: row[5] || undefined
      }));
  }
}
```

### Step 2: React Component

Create your main component in `src/components/BCMExplorer.tsx`:

```typescript
import * as React from 'react';
import { useState } from 'react';
import { WebPartContext } from '@microsoft/sp-webpart-base';
import { ExcelBCMService } from '../services/ExcelBCMService';
import { CapabilityApplication } from '../types/bcm-types';

type BCMExplorerProps = {
  context: WebPartContext;
  workbookId: string;
};

export const BCMExplorer: React.FC<BCMExplorerProps> = ({ context, workbookId }) => {
  const [selectedCapability, setSelectedCapability] = useState<string>('');
  const [applications, setApplications] = useState<CapabilityApplication[]>([]);
  const [loading, setLoading] = useState<boolean>(false);

  const excelService = new ExcelBCMService(context, workbookId);

  const handleCapabilitySearch = async () => {
    if (!selectedCapability) return;
    
    setLoading(true);
    try {
      const apps = await excelService.getApplicationsForCapability(selectedCapability);
      setApplications(apps);
    } catch (error) {
      console.error('Error loading applications:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ padding: '20px' }}>
      <h2>Business Capability Explorer</h2>
      
      <div style={{ marginBottom: '20px' }}>
        <input
          type="text"
          placeholder="Enter capability name (e.g., Asset Management)"
          value={selectedCapability}
          onChange={(e) => setSelectedCapability(e.target.value)}
          style={{ padding: '8px', marginRight: '10px', width: '300px' }}
        />
        <button 
          onClick={handleCapabilitySearch}
          disabled={loading}
          style={{ padding: '8px 16px' }}
        >
          {loading ? 'Loading...' : 'Find Applications'}
        </button>
      </div>

      {applications.length > 0 && (
        <div>
          <h3>Applications supporting "{selectedCapability}":</h3>
          <table style={{ borderCollapse: 'collapse', width: '100%' }}>
            <thead>
              <tr style={{ backgroundColor: '#f0f0f0' }}>
                <th style={{ border: '1px solid #ccc', padding: '8px' }}>Application</th>
                <th style={{ border: '1px solid #ccc', padding: '8px' }}>Usage Type</th>
                <th style={{ border: '1px solid #ccc', padding: '8px' }}>Notes</th>
              </tr>
            </thead>
            <tbody>
              {applications.map((app, index) => (
                <tr key={index}>
                  <td style={{ border: '1px solid #ccc', padding: '8px' }}>{app.applicationName}</td>
                  <td style={{ border: '1px solid #ccc', padding: '8px' }}>{app.usageType}</td>
                  <td style={{ border: '1px solid #ccc', padding: '8px' }}>{app.notes}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
};
```

## Getting Your Workbook ID

You need to find your Excel file's unique ID to connect to it:

### Method 1: Using Graph Explorer (Recommended)

1. **Go to**: https://developer.microsoft.com/en-us/graph/graph-explorer
2. **Sign in** with your Office 365 account  
3. **Run this query**: `GET /me/drive/search(q='BCM-Data.xlsx')`
4. **Copy the "id" field** from the results
5. **Use this ID** in your web part configuration

### Method 2: From SharePoint URL

If your Excel file is in SharePoint, you can extract the ID from the URL when you open the file.

## Web Part Configuration

Update your web part's main file to include the workbook ID:

```typescript
// In your main web part file (e.g., BcmExplorerWebPart.ts)
export interface IBcmExplorerWebPartProps {
  workbookId: string;
}

export default class BcmExplorerWebPart extends BaseClientSideWebPart<IBcmExplorerWebPartProps> {
  
  public render(): void {
    const element: React.ReactElement<BCMExplorerProps> = React.createElement(
      BCMExplorer,
      {
        context: this.context,
        workbookId: this.properties.workbookId
      }
    );

    ReactDom.render(element, this.domElement);
  }

  protected getPropertyPaneConfiguration(): IPropertyPaneConfiguration {
    return {
      pages: [
        {
          header: {
            description: "BCM Explorer Settings"
          },
          groups: [
            {
              groupName: "Excel Configuration", 
              groupFields: [
                PropertyPaneTextField('workbookId', {
                  label: "Excel Workbook ID",
                  description: "The ID of your BCM Excel workbook"
                })
              ]
            }
          ]
        }
      ]
    };
  }
}
```

## Deployment

### Build and Package

```bash
# Build for production
gulp bundle --ship
gulp package-solution --ship

# Upload the .sppkg file to your SharePoint App Catalog
```

### Configure the Web Part

1. **Add the web part** to a SharePoint page
2. **Edit the web part**
3. **Enter your workbook ID** in the settings
4. **Save the page**

## Usage

**For End Users**:
- Enter a capability name and click "Find Applications"
- See which applications support that capability
- View usage type (Primary vs Supporting)

**For Business Users**:
- Edit the Excel file to update relationships
- Changes appear automatically in the web part
- No need to redeploy or update anything

## Extending the System

Want to add more features? Here are simple additions:

**Search Applications**: Add a second input to find capabilities for an application
**Hierarchy View**: Show parent/child capability relationships  
**Better UI**: Use Fluent UI components for a more polished look
**Bulk Display**: Show all data in searchable tables

The beauty of this approach is that all the complex relationship logic is handled by simple Excel filtering - your React code stays minimal and focused on display.

## Troubleshooting

**Web part shows "Loading..." forever**:
- Check that the workbook ID is correct
- Verify you have access to the Excel file
- Check browser console for error messages

**"Application not found" errors**:
- Ensure capability names match exactly (case-sensitive)
- Check that your Excel sheets have the correct column headers
- Verify there are no extra spaces in your Excel data

**Changes in Excel don't appear**:
- Excel Online updates can take 1-2 minutes to sync
- Try refreshing the SharePoint page
- Check that you saved the Excel file

The system is designed to be simple and reliable - most issues are data formatting problems that are easy to fix in Excel.