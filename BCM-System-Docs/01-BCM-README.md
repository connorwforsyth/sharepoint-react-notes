# Business Capability Model System
**📖 Page 1 of 5 | System Overview**

A simple, zero-maintenance system for managing business capabilities and application relationships using Excel Online and SharePoint Framework.

---
**Navigation:** [📋 Table of Contents](./00-Table-of-Contents.md) | **Page 1** | [▶️ Next: Quick Start](./02-BCM-Getting-Started.md)  
---

## What This System Does

**For Business Users**:
- Manage capability-application relationships in familiar Excel interface
- Bulk edit relationships using copy/paste and standard Excel features
- No technical knowledge required for data updates

**For End Users**:  
- Explore which applications support each business capability
- Navigate capability hierarchies through SharePoint web part
- Search and filter relationships easily

**For IT**:
- Zero ongoing maintenance once deployed
- No database administration required
- Uses existing Office 365 infrastructure

## Key Benefits

✅ **Zero Maintenance** - System runs itself after setup  
✅ **Unlimited Scale** - No database limits on relationships  
✅ **Business User Control** - Data managed by business, not IT  
✅ **Familiar Tools** - Excel for data, SharePoint for access  
✅ **Real-time Updates** - Changes appear automatically  
✅ **Cost Effective** - Uses existing Office 365 licenses  

## Architecture

**Data Storage**: Excel Online workbook (3 sheets)
- Capabilities sheet: Business capabilities with hierarchy
- Applications sheet: Software applications catalog  
- Relationships sheet: Which apps support which capabilities

**User Interface**: SharePoint Framework web part
- Reads Excel data via Microsoft Graph API
- Provides search and exploration features
- Updates automatically when Excel changes

**No Database Required**: Excel Online handles all data management, relationships, backup, versioning, and concurrent access.

## Getting Started

### For Business Users
1. **[Excel Setup Guide](./03-BCM-Excel-Setup.md)** - Set up your data in Excel Online
2. **[Quick Reference](./05-BCM-Quick-Reference.md)** - Day-to-day usage and data management

### For Developers  
1. **[Getting Started](./02-BCM-Getting-Started.md)** - System overview and architecture
2. **[Web Part Guide](./04-BCM-Web-Part-Guide.md)** - Build the SharePoint interface

### For Everyone
- **[Quick Reference](./05-BCM-Quick-Reference.md)** - Essential information for all users

## Example Use Cases

**"Which applications does Asset Management use?"**
- Business user: Maintains relationships in Excel
- End user: Searches via SharePoint web part
- Result: See Power BI (Supporting), SAP PM (Primary), etc.

**"Which capabilities use Power BI?"** 
- Search Power BI in web part
- See all capabilities: Asset Management, Network Operations, Customer Analytics
- Understand usage type: Primary vs Supporting

**"Add new application relationship"**
- Business user opens Excel
- Adds row: "Customer Management | New CRM | Primary"  
- Saves Excel file
- Change appears in web part within minutes

## Why Excel Online?

**Traditional database approaches**:
❌ Complex setup and maintenance  
❌ Requires database admin skills  
❌ 5,000 item limits in SharePoint Lists  
❌ Technical users needed for updates  
❌ Expensive to scale and maintain  

**Excel Online approach**:
✅ Familiar interface everyone knows  
✅ Unlimited relationships and scale  
✅ Business users manage their own data  
✅ Zero maintenance overhead  
✅ Built-in backup, versioning, collaboration  

## Technical Requirements

**For Setup**:
- SharePoint Framework development environment
- Access to SharePoint App Catalog for deployment
- Excel Online workbook in SharePoint or OneDrive

**For Usage**:
- Office 365 with SharePoint Online
- Modern SharePoint pages
- Standard user permissions

**No Additional Licensing**: Uses existing Office 365 and SharePoint capabilities.

## Support and Maintenance

**Ongoing Maintenance**: None required - system is self-managing

**User Support**:
- Business users: Standard Excel Online support
- End users: Standard SharePoint web part support  
- Developers: Microsoft Graph API and SPFx documentation

**Data Issues**: Business users can resolve directly in Excel - no IT intervention needed.

## Success Stories

This approach has proven successful for organizations that need:
- **Capability mapping** across large enterprises
- **Application portfolio management** 
- **Business-IT alignment** initiatives
- **Zero-maintenance solutions** 
- **Business user empowerment** for data management

The combination of Excel's flexibility with SharePoint's access control creates a powerful, maintenance-free system that scales with your organization.