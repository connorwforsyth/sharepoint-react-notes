# SharePoint Lists as Relational Database

This guide provides comprehensive patterns for implementing relational database functionality using SharePoint Lists, including spreadsheet import strategies and advanced relationship management.

## Core Concepts for Relational SharePoint Lists

### Understanding SharePoint's Limitations and Strengths

SharePoint Lists are not traditional databases, but they can effectively model relational data when designed properly:

**Strengths:**
- Built-in security and permissions model
- Automatic audit trail (Created/Modified fields)
- Rich field types including lookups and people fields  
- REST API for programmatic access
- Excel-like interface for users
- Version history and workflows

**Limitations:**
- 5,000 item view threshold (requires indexing strategy)
- No foreign key constraints
- Limited join capabilities
- No cascading deletes (must implement in code)
- List relationships are lookup-based, not true foreign keys

## Designing Relational Structures

### 1. Entity Relationship Planning

Before creating lists, map your entities and relationships:

```typescript
// Example: Project Management System
type EntityRelationships = {
  // One-to-Many: One project has many tasks
  Projects: {
    primaryKey: "Id";
    relationships: {
      tasks: { type: "one-to-many"; foreignList: "Tasks"; foreignKey: "ProjectId" };
      milestones: { type: "one-to-many"; foreignList: "Milestones"; foreignKey: "ProjectId" };
    };
  };
  
  // Many-to-One: Many tasks belong to one project
  Tasks: {
    primaryKey: "Id";
    relationships: {
      project: { type: "many-to-one"; foreignList: "Projects"; lookupField: "ProjectId" };
      assignee: { type: "many-to-one"; foreignList: "Users"; lookupField: "AssignedToId" };
    };
  };
  
  // Many-to-Many: Projects can have multiple team members, users can be on multiple projects
  ProjectTeamMembers: {
    primaryKey: "Id";
    relationships: {
      project: { type: "many-to-one"; foreignList: "Projects"; lookupField: "ProjectId" };
      teamMember: { type: "many-to-one"; foreignList: "Users"; lookupField: "TeamMemberId" };
    };
  };
};
```

### 2. List Schema Design Patterns

```typescript
// src/types/relational-schemas.ts

// Base types for relational integrity
export type RelationalBaseItem = SharePointBaseItem & {
  // Custom fields for relationship tracking
  EntityVersion?: string; // For optimistic locking
  ParentEntityId?: number; // Generic parent reference
  SortOrder?: number; // For ordered relationships
};

// Master entity: Projects
export type ProjectEntity = RelationalBaseItem & {
  ProjectCode: string; // Business key for external references
  ProjectName: string;
  Description: string;
  StartDate: string;
  EndDate: string;
  Budget: number;
  Status: "Planning" | "Active" | "On Hold" | "Completed" | "Cancelled";
  Priority: "Low" | "Medium" | "High" | "Critical";
  ProjectManagerId: number; // User lookup
  ClientId?: number; // Optional client lookup
  // Calculated fields (populated by code)
  TaskCount?: number;
  CompletedTaskCount?: number;
  TotalEstimatedHours?: number;
  TotalActualHours?: number;
  ProgressPercentage?: number;
};

// Child entity: Tasks
export type TaskEntity = RelationalBaseItem & {
  TaskCode: string; // Business key
  TaskName: string;
  Description: string;
  ProjectId: number; // Foreign key to Projects
  ParentTaskId?: number; // Self-referencing for subtasks
  AssignedToId?: number; // User lookup
  Status: "Not Started" | "In Progress" | "Completed" | "Blocked";
  Priority: "Low" | "Medium" | "High" | "Critical";
  EstimatedHours: number;
  ActualHours: number;
  StartDate?: string;
  DueDate?: string;
  CompletedDate?: string;
  // Hierarchical fields
  TaskLevel: number; // 0 = parent, 1 = child, etc.
  TaskPath: string; // e.g., "1.2.3" for hierarchy
};

// Junction table for Many-to-Many relationships
export type ProjectTeamMemberEntity = RelationalBaseItem & {
  ProjectId: number;
  TeamMemberId: number;
  Role: "Project Manager" | "Developer" | "Designer" | "Tester" | "Business Analyst";
  AllocationPercentage: number; // 0-100
  StartDate: string;
  EndDate?: string;
  IsActive: boolean;
};

// Reference/Lookup entities
export type ClientEntity = RelationalBaseItem & {
  ClientCode: string;
  ClientName: string;
  ContactEmail: string;
  ContactPhone?: string;
  Address?: string;
  IsActive: boolean;
};

export type CategoryEntity = RelationalBaseItem & {
  CategoryCode: string;
  CategoryName: string;
  ParentCategoryId?: number; // Self-referencing hierarchy
  Description?: string;
  Color?: string; // For UI display
  IsActive: boolean;
};
```

### 3. List Definition with Relationships

```typescript
// src/lib/relational-list-definitions.ts
import { ListDefinition, ListFieldDefinition } from "./listSetup";

// Enhanced field definition for relationships
type RelationalFieldDefinition = ListFieldDefinition & {
  relationshipType?: "lookup" | "user" | "choice";
  lookupList?: string;
  lookupField?: string;
  allowMultipleValues?: boolean;
  indexed?: boolean; // Critical for performance
};

type RelationalListDefinition = {
  title: string;
  description: string;
  template: number;
  fields: RelationalFieldDefinition[];
  indexes: string[]; // Fields that need indexing for performance
  relationships: {
    type: "one-to-many" | "many-to-one" | "many-to-many";
    relatedList: string;
    foreignKey?: string;
    junctionList?: string; // For many-to-many
  }[];
};

export const projectsListDefinition: RelationalListDefinition = {
  title: "Projects",
  description: "Master project list with relational integrity",
  template: 100,
  fields: [
    { name: "ProjectCode", type: "Text", required: true, indexed: true },
    { name: "ProjectName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    { name: "StartDate", type: "DateTime", required: true, indexed: true },
    { name: "EndDate", type: "DateTime", indexed: true },
    { name: "Budget", type: "Number" },
    { 
      name: "Status", 
      type: "Choice", 
      choices: ["Planning", "Active", "On Hold", "Completed", "Cancelled"],
      indexed: true 
    },
    { 
      name: "Priority", 
      type: "Choice", 
      choices: ["Low", "Medium", "High", "Critical"],
      indexed: true 
    },
    { 
      name: "ProjectManager", 
      type: "User", 
      required: true,
      relationshipType: "user",
      indexed: true 
    },
    { 
      name: "Client", 
      type: "Lookup", 
      relationshipType: "lookup",
      lookupList: "Clients",
      lookupField: "ClientName",
      indexed: true 
    },
    // Calculated fields (updated by code)
    { name: "TaskCount", type: "Number" },
    { name: "CompletedTaskCount", type: "Number" },
    { name: "TotalEstimatedHours", type: "Number" },
    { name: "TotalActualHours", type: "Number" },
    { name: "ProgressPercentage", type: "Number" },
  ],
  indexes: ["ProjectCode", "Status", "StartDate", "ProjectManager", "Client"],
  relationships: [
    { type: "one-to-many", relatedList: "Tasks", foreignKey: "ProjectId" },
    { type: "one-to-many", relatedList: "Milestones", foreignKey: "ProjectId" },
    { type: "many-to-many", relatedList: "Users", junctionList: "ProjectTeamMembers" },
  ],
};

export const tasksListDefinition: RelationalListDefinition = {
  title: "Tasks",
  description: "Project tasks with hierarchical support",
  template: 100,
  fields: [
    { name: "TaskCode", type: "Text", required: true, indexed: true },
    { name: "TaskName", type: "Text", required: true },
    { name: "Description", type: "Text" },
    { 
      name: "Project", 
      type: "Lookup", 
      required: true,
      relationshipType: "lookup",
      lookupList: "Projects",
      lookupField: "ProjectName",
      indexed: true 
    },
    { 
      name: "ParentTask", 
      type: "Lookup", 
      relationshipType: "lookup",
      lookupList: "Tasks",
      lookupField: "TaskName" 
    },
    { 
      name: "AssignedTo", 
      type: "User",
      relationshipType: "user",
      indexed: true 
    },
    { 
      name: "Status", 
      type: "Choice", 
      choices: ["Not Started", "In Progress", "Completed", "Blocked"],
      indexed: true 
    },
    { 
      name: "Priority", 
      type: "Choice", 
      choices: ["Low", "Medium", "High", "Critical"],
      indexed: true 
    },
    { name: "EstimatedHours", type: "Number", required: true },
    { name: "ActualHours", type: "Number" },
    { name: "StartDate", type: "DateTime", indexed: true },
    { name: "DueDate", type: "DateTime", indexed: true },
    { name: "CompletedDate", type: "DateTime" },
    { name: "TaskLevel", type: "Number" },
    { name: "TaskPath", type: "Text" },
  ],
  indexes: ["TaskCode", "Project", "AssignedTo", "Status", "DueDate"],
  relationships: [
    { type: "many-to-one", relatedList: "Projects", foreignKey: "ProjectId" },
    { type: "many-to-one", relatedList: "Tasks", foreignKey: "ParentTaskId" },
  ],
};

export const projectTeamMembersListDefinition: RelationalListDefinition = {
  title: "ProjectTeamMembers",
  description: "Junction table for project-user many-to-many relationships",
  template: 100,
  fields: [
    { 
      name: "Project", 
      type: "Lookup", 
      required: true,
      relationshipType: "lookup",
      lookupList: "Projects",
      lookupField: "ProjectName",
      indexed: true 
    },
    { 
      name: "TeamMember", 
      type: "User", 
      required: true,
      relationshipType: "user",
      indexed: true 
    },
    { 
      name: "Role", 
      type: "Choice", 
      choices: ["Project Manager", "Developer", "Designer", "Tester", "Business Analyst"],
      required: true 
    },
    { name: "AllocationPercentage", type: "Number", required: true },
    { name: "StartDate", type: "DateTime", required: true },
    { name: "EndDate", type: "DateTime" },
    { name: "IsActive", type: "Boolean" },
  ],
  indexes: ["Project", "TeamMember", "IsActive"],
  relationships: [
    { type: "many-to-one", relatedList: "Projects", foreignKey: "ProjectId" },
    { type: "many-to-one", relatedList: "Users", foreignKey: "TeamMemberId" },
  ],
};
```

## Spreadsheet Import Strategies

### 1. Pre-Import Data Validation and Preparation

```typescript
// src/lib/spreadsheet-import.ts
import * as XLSX from 'xlsx';

type ImportValidationResult = {
  isValid: boolean;
  errors: string[];
  warnings: string[];
  data: any[];
  summary: {
    totalRows: number;
    validRows: number;
    errorRows: number;
  };
};

type ImportMapping = {
  sourceColumn: string;
  targetField: string;
  fieldType: "Text" | "Number" | "DateTime" | "Boolean" | "Choice" | "Lookup" | "User";
  required: boolean;
  validation?: (value: any) => boolean;
  transform?: (value: any) => any;
  lookupList?: string;
  lookupField?: string;
};

export class SpreadsheetImportService {
  private context: WebPartContext;
  private sharePointService: SharePointService;

  constructor(context: WebPartContext) {
    this.context = context;
    this.sharePointService = new SharePointService(context);
  }

  // Parse Excel/CSV file
  async parseSpreadsheetFile(file: File): Promise<any[]> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      
      reader.onload = (e) => {
        try {
          const data = new Uint8Array(e.target?.result as ArrayBuffer);
          const workbook = XLSX.read(data, { type: 'array' });
          
          // Get first worksheet
          const worksheetName = workbook.SheetNames[0];
          const worksheet = workbook.Sheets[worksheetName];
          
          // Convert to JSON
          const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });
          
          resolve(jsonData);
        } catch (error) {
          reject(error);
        }
      };
      
      reader.onerror = () => reject(new Error('Failed to read file'));
      reader.readAsArrayBuffer(file);
    });
  }

  // Validate and transform data according to mapping
  async validateImportData(
    rawData: any[], 
    mapping: ImportMapping[],
    startRow: number = 1 // Skip header row
  ): Promise<ImportValidationResult> {
    const errors: string[] = [];
    const warnings: string[] = [];
    const validatedData: any[] = [];
    
    // Skip header row
    const dataRows = rawData.slice(startRow);
    
    for (let rowIndex = 0; rowIndex < dataRows.length; rowIndex++) {
      const row = dataRows[rowIndex];
      const rowNumber = rowIndex + startRow + 1; // +1 for 1-based indexing
      const validatedRow: any = {};
      let rowIsValid = true;
      
      // Process each field mapping
      for (const fieldMapping of mapping) {
        const sourceColumnIndex = this.getColumnIndex(rawData[0], fieldMapping.sourceColumn);
        
        if (sourceColumnIndex === -1) {
          errors.push(`Row ${rowNumber}: Source column '${fieldMapping.sourceColumn}' not found`);
          rowIsValid = false;
          continue;
        }
        
        let cellValue = row[sourceColumnIndex];
        
        // Handle empty values
        if (cellValue === undefined || cellValue === null || cellValue === '') {
          if (fieldMapping.required) {
            errors.push(`Row ${rowNumber}: Required field '${fieldMapping.targetField}' is empty`);
            rowIsValid = false;
            continue;
          } else {
            validatedRow[fieldMapping.targetField] = null;
            continue;
          }
        }
        
        // Transform value if needed
        if (fieldMapping.transform) {
          try {
            cellValue = fieldMapping.transform(cellValue);
          } catch (error) {
            errors.push(`Row ${rowNumber}: Transform error for '${fieldMapping.targetField}': ${error.message}`);
            rowIsValid = false;
            continue;
          }
        }
        
        // Validate by field type
        const validationResult = await this.validateFieldValue(cellValue, fieldMapping, rowNumber);
        
        if (!validationResult.isValid) {
          errors.push(...validationResult.errors);
          rowIsValid = false;
          continue;
        }
        
        if (validationResult.warnings.length > 0) {
          warnings.push(...validationResult.warnings);
        }
        
        validatedRow[fieldMapping.targetField] = validationResult.value;
      }
      
      if (rowIsValid) {
        validatedData.push(validatedRow);
      }
    }
    
    return {
      isValid: errors.length === 0,
      errors,
      warnings,
      data: validatedData,
      summary: {
        totalRows: dataRows.length,
        validRows: validatedData.length,
        errorRows: dataRows.length - validatedData.length,
      },
    };
  }

  // Validate individual field values
  private async validateFieldValue(
    value: any, 
    mapping: ImportMapping, 
    rowNumber: number
  ): Promise<{ isValid: boolean; errors: string[]; warnings: string[]; value: any }> {
    const errors: string[] = [];
    const warnings: string[] = [];
    let processedValue = value;
    
    switch (mapping.fieldType) {
      case "Text":
        processedValue = String(value);
        if (processedValue.length > 255) {
          warnings.push(`Row ${rowNumber}: Text field '${mapping.targetField}' truncated to 255 characters`);
          processedValue = processedValue.substring(0, 255);
        }
        break;
        
      case "Number":
        processedValue = Number(value);
        if (isNaN(processedValue)) {
          errors.push(`Row ${rowNumber}: Invalid number for field '${mapping.targetField}': ${value}`);
          return { isValid: false, errors, warnings, value: null };
        }
        break;
        
      case "DateTime":
        // Handle various date formats
        const dateValue = this.parseDate(value);
        if (!dateValue) {
          errors.push(`Row ${rowNumber}: Invalid date format for field '${mapping.targetField}': ${value}`);
          return { isValid: false, errors, warnings, value: null };
        }
        processedValue = dateValue.toISOString();
        break;
        
      case "Boolean":
        processedValue = this.parseBoolean(value);
        if (processedValue === null) {
          errors.push(`Row ${rowNumber}: Invalid boolean value for field '${mapping.targetField}': ${value}`);
          return { isValid: false, errors, warnings, value: null };
        }
        break;
        
      case "Choice":
        // Validate against SharePoint choice field options
        const choiceValidation = await this.validateChoiceField(value, mapping, rowNumber);
        if (!choiceValidation.isValid) {
          errors.push(...choiceValidation.errors);
          return { isValid: false, errors, warnings, value: null };
        }
        processedValue = choiceValidation.value;
        break;
        
      case "Lookup":
        // Validate lookup value exists in target list
        const lookupValidation = await this.validateLookupField(value, mapping, rowNumber);
        if (!lookupValidation.isValid) {
          errors.push(...lookupValidation.errors);
          return { isValid: false, errors, warnings, value: null };
        }
        processedValue = lookupValidation.value;
        break;
        
      case "User":
        // Validate user exists
        const userValidation = await this.validateUserField(value, rowNumber);
        if (!userValidation.isValid) {
          errors.push(...userValidation.errors);
          return { isValid: false, errors, warnings, value: null };
        }
        processedValue = userValidation.value;
        break;
    }
    
    // Custom validation if provided
    if (mapping.validation && !mapping.validation(processedValue)) {
      errors.push(`Row ${rowNumber}: Custom validation failed for field '${mapping.targetField}'`);
      return { isValid: false, errors, warnings, value: null };
    }
    
    return { isValid: true, errors, warnings, value: processedValue };
  }

  // Helper methods for validation
  private getColumnIndex(headerRow: any[], columnName: string): number {
    if (!headerRow) return -1;
    
    return headerRow.findIndex((header: any) => 
      String(header).toLowerCase().trim() === columnName.toLowerCase().trim()
    );
  }

  private parseDate(value: any): Date | null {
    if (!value) return null;
    
    // Try various date formats
    const dateFormats = [
      // Excel date serial number
      () => {
        const num = Number(value);
        if (!isNaN(num) && num > 1) {
          // Excel date serial number (1 = Jan 1, 1900)
          return new Date((num - 25569) * 86400 * 1000);
        }
        return null;
      },
      // Standard formats
      () => new Date(value),
      // Custom formats
      () => {
        const str = String(value);
        // Handle MM/DD/YYYY
        const mmddyyyy = str.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})$/);
        if (mmddyyyy) {
          return new Date(parseInt(mmddyyyy[3]), parseInt(mmddyyyy[1]) - 1, parseInt(mmddyyyy[2]));
        }
        // Handle DD/MM/YYYY
        const ddmmyyyy = str.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})$/);
        if (ddmmyyyy) {
          return new Date(parseInt(ddmmyyyy[3]), parseInt(ddmmyyyy[2]) - 1, parseInt(ddmmyyyy[1]));
        }
        return null;
      },
    ];
    
    for (const parseFormat of dateFormats) {
      try {
        const date = parseFormat();
        if (date && !isNaN(date.getTime())) {
          return date;
        }
      } catch {
        continue;
      }
    }
    
    return null;
  }

  private parseBoolean(value: any): boolean | null {
    if (typeof value === 'boolean') return value;
    
    const str = String(value).toLowerCase().trim();
    
    if (['true', '1', 'yes', 'y', 'on'].includes(str)) return true;
    if (['false', '0', 'no', 'n', 'off'].includes(str)) return false;
    
    return null;
  }

  private async validateChoiceField(
    value: any, 
    mapping: ImportMapping, 
    rowNumber: number
  ): Promise<{ isValid: boolean; errors: string[]; value: any }> {
    // This would need to fetch the actual choice options from SharePoint
    // For now, implement basic validation
    const stringValue = String(value).trim();
    
    // TODO: Fetch actual choice options from SharePoint field
    // const field = await this.sharePointService.getFieldChoices(listName, mapping.targetField);
    
    return { isValid: true, errors: [], value: stringValue };
  }

  private async validateLookupField(
    value: any, 
    mapping: ImportMapping, 
    rowNumber: number
  ): Promise<{ isValid: boolean; errors: string[]; value: any }> {
    if (!mapping.lookupList || !mapping.lookupField) {
      return { 
        isValid: false, 
        errors: [`Row ${rowNumber}: Lookup field configuration missing for '${mapping.targetField}'`], 
        value: null 
      };
    }
    
    try {
      // Search for the lookup value in the target list
      const lookupItems = await this.sharePointService.getListItems(
        mapping.lookupList,
        ["Id", mapping.lookupField],
        [],
        `${mapping.lookupField} eq '${String(value)}'`
      );
      
      if (lookupItems.length === 0) {
        return { 
          isValid: false, 
          errors: [`Row ${rowNumber}: Lookup value '${value}' not found in list '${mapping.lookupList}'`], 
          value: null 
        };
      }
      
      if (lookupItems.length > 1) {
        // Use first match but warn about duplicates
        return { 
          isValid: true, 
          errors: [], 
          value: lookupItems[0].Id 
        };
      }
      
      return { isValid: true, errors: [], value: lookupItems[0].Id };
      
    } catch (error) {
      return { 
        isValid: false, 
        errors: [`Row ${rowNumber}: Error validating lookup field '${mapping.targetField}': ${error.message}`], 
        value: null 
      };
    }
  }

  private async validateUserField(
    value: any, 
    rowNumber: number
  ): Promise<{ isValid: boolean; errors: string[]; value: any }> {
    const email = String(value).trim();
    
    // Basic email validation
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      return { 
        isValid: false, 
        errors: [`Row ${rowNumber}: Invalid email format: ${email}`], 
        value: null 
      };
    }
    
    try {
      // TODO: Validate user exists in SharePoint
      // const user = await this.sharePointService.ensureUser(email);
      // return { isValid: true, errors: [], value: user.Id };
      
      // For now, return email (SharePoint will resolve)
      return { isValid: true, errors: [], value: email };
      
    } catch (error) {
      return { 
        isValid: false, 
        errors: [`Row ${rowNumber}: User not found: ${email}`], 
        value: null 
      };
    }
  }

  // Import validated data to SharePoint
  async importToSharePoint(
    listName: string,
    validatedData: any[],
    batchSize: number = 50,
    onProgress?: (completed: number, total: number) => void
  ): Promise<{ success: number; failed: number; errors: string[] }> {
    const results = { success: 0, failed: 0, errors: [] };
    const batches = this.chunkArray(validatedData, batchSize);
    
    let completed = 0;
    
    for (const batch of batches) {
      try {
        await this.sharePointService.batchCreateItems(listName, batch);
        results.success += batch.length;
        completed += batch.length;
        
        if (onProgress) {
          onProgress(completed, validatedData.length);
        }
      } catch (error) {
        results.failed += batch.length;
        results.errors.push(`Batch import failed: ${error.message}`);
        completed += batch.length;
        
        if (onProgress) {
          onProgress(completed, validatedData.length);
        }
      }
    }
    
    return results;
  }

  private chunkArray<T>(array: T[], chunkSize: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += chunkSize) {
      chunks.push(array.slice(i, i + chunkSize));
    }
    return chunks;
  }
}
```

### 2. Import Mapping Configuration

```typescript
// src/lib/import-mappings.ts

// Pre-defined mappings for common scenarios
export const projectImportMapping: ImportMapping[] = [
  {
    sourceColumn: "Project Code",
    targetField: "ProjectCode",
    fieldType: "Text",
    required: true,
    validation: (value: string) => value.length >= 3 && value.length <= 10,
  },
  {
    sourceColumn: "Project Name",
    targetField: "Title", // Maps to SharePoint's Title field
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
    sourceColumn: "Start Date",
    targetField: "StartDate",
    fieldType: "DateTime",
    required: true,
  },
  {
    sourceColumn: "End Date",
    targetField: "EndDate",
    fieldType: "DateTime",
    required: false,
  },
  {
    sourceColumn: "Budget",
    targetField: "Budget",
    fieldType: "Number",
    required: false,
    transform: (value: any) => {
      // Remove currency symbols and commas
      const cleaned = String(value).replace(/[$,]/g, '');
      return Number(cleaned);
    },
  },
  {
    sourceColumn: "Status", 
    targetField: "Status",
    fieldType: "Choice",
    required: true,
    transform: (value: any) => {
      // Normalize status values
      const statusMap: { [key: string]: string } = {
        "planning": "Planning",
        "active": "Active",
        "in progress": "Active",
        "on hold": "On Hold",
        "hold": "On Hold",
        "completed": "Completed",
        "done": "Completed",
        "cancelled": "Cancelled",
        "canceled": "Cancelled",
      };
      
      const normalized = String(value).toLowerCase().trim();
      return statusMap[normalized] || value;
    },
  },
  {
    sourceColumn: "Project Manager Email",
    targetField: "ProjectManager",
    fieldType: "User",
    required: true,
  },
  {
    sourceColumn: "Client Name",
    targetField: "Client",
    fieldType: "Lookup",
    required: false,
    lookupList: "Clients",
    lookupField: "ClientName",
  },
];

export const taskImportMapping: ImportMapping[] = [
  {
    sourceColumn: "Task Code",
    targetField: "TaskCode",
    fieldType: "Text",
    required: true,
  },
  {
    sourceColumn: "Task Name",
    targetField: "Title",
    fieldType: "Text",
    required: true,
  },
  {
    sourceColumn: "Project Code", // Will be resolved to Project lookup
    targetField: "Project",
    fieldType: "Lookup",
    required: true,
    lookupList: "Projects",
    lookupField: "ProjectCode",
  },
  {
    sourceColumn: "Assigned To Email",
    targetField: "AssignedTo",
    fieldType: "User",
    required: false,
  },
  {
    sourceColumn: "Estimated Hours",
    targetField: "EstimatedHours",
    fieldType: "Number",
    required: true,
    transform: (value: any) => Math.max(0, Number(value) || 0),
  },
  {
    sourceColumn: "Due Date",
    targetField: "DueDate",
    fieldType: "DateTime",
    required: false,
  },
];
```

## Advanced Relational Operations

### 1. Complex Query Service for Related Data

```typescript
// src/lib/relational-query-service.ts
export class RelationalQueryService extends SharePointService {
  
  // Get project with all related data
  async getProjectWithAllRelations(projectId: number): Promise<{
    project: ProjectEntity;
    tasks: TaskEntity[];
    teamMembers: ProjectTeamMemberEntity[];
    milestones: any[]; // Define milestone type
    completionStats: {
      totalTasks: number;
      completedTasks: number;
      progressPercentage: number;
      totalEstimatedHours: number;
      totalActualHours: number;
    };
  }> {
    
    // Execute all queries in parallel
    const [project, tasks, teamMembers] = await Promise.all([
      this.getListItemById<ProjectEntity>("Projects", projectId),
      this.getListItems<TaskEntity>(
        "Tasks",
        ["*", "AssignedTo/Title", "AssignedTo/Email"],
        ["AssignedTo"],
        `Project/Id eq ${projectId}`
      ),
      this.getListItems<ProjectTeamMemberEntity>(
        "ProjectTeamMembers",
        ["*", "TeamMember/Title", "TeamMember/Email"],
        ["TeamMember"],
        `Project/Id eq ${projectId} and IsActive eq true`
      ),
    ]);
    
    // Calculate completion stats
    const completedTasks = tasks.filter(t => t.Status === "Completed");
    const completionStats = {
      totalTasks: tasks.length,
      completedTasks: completedTasks.length,
      progressPercentage: tasks.length > 0 ? Math.round((completedTasks.length / tasks.length) * 100) : 0,
      totalEstimatedHours: tasks.reduce((sum, t) => sum + (t.EstimatedHours || 0), 0),
      totalActualHours: tasks.reduce((sum, t) => sum + (t.ActualHours || 0), 0),
    };
    
    return {
      project,
      tasks,
      teamMembers,
      milestones: [], // TODO: Implement milestones
      completionStats,
    };
  }
  
  // Get user workload across all projects
  async getUserWorkloadAnalysis(userId: number): Promise<{
    activeProjects: ProjectEntity[];
    assignedTasks: TaskEntity[];
    teamMemberships: ProjectTeamMemberEntity[];
    workloadSummary: {
      totalProjects: number;
      totalTasks: number;
      overdueTasks: number;
      totalAllocationPercentage: number;
      estimatedHoursThisWeek: number;
    };
  }> {
    
    const [assignedTasks, teamMemberships] = await Promise.all([
      this.getListItems<TaskEntity>(
        "Tasks",
        ["*", "Project/Title", "Project/Id"],
        ["Project"],
        `AssignedTo/Id eq ${userId} and Status ne 'Completed'`
      ),
      this.getListItems<ProjectTeamMemberEntity>(
        "ProjectTeamMembers",
        ["*", "Project/Title", "Project/Id"],
        ["Project"],
        `TeamMember/Id eq ${userId} and IsActive eq true`
      ),
    ]);
    
    // Get unique active projects
    const projectIds = [...new Set([
      ...assignedTasks.map(t => t.ProjectId),
      ...teamMemberships.map(tm => tm.ProjectId),
    ])];
    
    const activeProjects = await Promise.all(
      projectIds.map(id => this.getListItemById<ProjectEntity>("Projects", id))
    );
    
    // Calculate workload metrics
    const now = new Date();
    const overdueTasks = assignedTasks.filter(
      t => t.DueDate && new Date(t.DueDate) < now
    );
    
    const workloadSummary = {
      totalProjects: activeProjects.length,
      totalTasks: assignedTasks.length,
      overdueTasks: overdueTasks.length,
      totalAllocationPercentage: teamMemberships.reduce(
        (sum, tm) => sum + (tm.AllocationPercentage || 0), 0
      ),
      estimatedHoursThisWeek: this.calculateWeeklyHours(assignedTasks),
    };
    
    return {
      activeProjects,
      assignedTasks,
      teamMemberships,
      workloadSummary,
    };
  }
  
  private calculateWeeklyHours(tasks: TaskEntity[]): number {
    const now = new Date();
    const weekStart = new Date(now.setDate(now.getDate() - now.getDay()));
    const weekEnd = new Date(weekStart);
    weekEnd.setDate(weekEnd.getDate() + 7);
    
    return tasks
      .filter(t => {
        if (!t.DueDate) return false;
        const dueDate = new Date(t.DueDate);
        return dueDate >= weekStart && dueDate <= weekEnd;
      })
      .reduce((sum, t) => sum + (t.EstimatedHours || 0), 0);
  }
  
  // Maintain referential integrity
  async deleteProjectCascade(projectId: number): Promise<void> {
    // Get all related items first
    const [tasks, teamMembers] = await Promise.all([
      this.getListItems<TaskEntity>("Tasks", ["Id"], [], `Project/Id eq ${projectId}`),
      this.getListItems<ProjectTeamMemberEntity>("ProjectTeamMembers", ["Id"], [], `Project/Id eq ${projectId}`),
    ]);
    
    // Delete in reverse dependency order
    await Promise.all([
      // Delete team memberships
      ...teamMembers.map(tm => this.deleteListItem("ProjectTeamMembers", tm.Id)),
      // Delete tasks
      ...tasks.map(task => this.deleteListItem("Tasks", task.Id)),
    ]);
    
    // Finally delete the project
    await this.deleteListItem("Projects", projectId);
  }
  
  // Update calculated fields
  async updateProjectCalculatedFields(projectId: number): Promise<void> {
    const tasks = await this.getListItems<TaskEntity>(
      "Tasks",
      ["Status", "EstimatedHours", "ActualHours"],
      [],
      `Project/Id eq ${projectId}`
    );
    
    const completedTasks = tasks.filter(t => t.Status === "Completed");
    const calculatedFields = {
      TaskCount: tasks.length,
      CompletedTaskCount: completedTasks.length,
      TotalEstimatedHours: tasks.reduce((sum, t) => sum + (t.EstimatedHours || 0), 0),
      TotalActualHours: tasks.reduce((sum, t) => sum + (t.ActualHours || 0), 0),
      ProgressPercentage: tasks.length > 0 ? Math.round((completedTasks.length / tasks.length) * 100) : 0,
    };
    
    await this.updateListItem("Projects", projectId, calculatedFields);
  }
}
```

Please share your IA image so I can provide more specific guidance tailored to your exact requirements!

<function_calls>
<invoke name="TodoWrite">
<parameter name="todos">[{"id": "1", "content": "Write comprehensive SharePoint Lists relational database guidance", "status": "completed", "priority": "high"}, {"id": "2", "content": "Review IA image when provided by user", "status": "pending", "priority": "high"}, {"id": "3", "content": "Create guidance for importing spreadsheets to SharePoint Lists", "status": "completed", "priority": "high"}, {"id": "4", "content": "Update existing SharePoint lists documentation with relational patterns", "status": "pending", "priority": "medium"}]