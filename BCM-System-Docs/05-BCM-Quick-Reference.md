# BCM System Quick Reference
**📖 Page 5 of 5 | Quick Reference**

Essential information for using and maintaining your Business Capability Model system.

---
**Navigation:** [📋 Table of Contents](./00-Table-of-Contents.md) | [◀️ Previous: Development Guide](./04-BCM-Web-Part-Guide.md) | **Page 5**  
---

## System Overview

**What it does**: Manages relationships between business capabilities and software applications  
**Data storage**: Excel Online workbook (3 sheets)  
**User interface**: SharePoint web part  
**Maintenance**: Zero - it's self-managing  

## For Business Users (Data Managers)

### Making Changes

**To add a new capability**:
1. Open the Excel workbook
2. Go to "Capabilities" sheet  
3. Add new row with capability details
4. Save (auto-saves in Excel Online)

**To add application relationships**:
1. Go to "CapabilityApplications" sheet
2. Add row: CapabilityName | ApplicationName | UsageType
3. Save
4. Changes appear in web part within 1-2 minutes

**To bulk update relationships**:
1. Use Excel's copy/paste, fill-down, and filter features
2. Import from other spreadsheets 
3. Use Find & Replace for bulk changes

### Data Rules

✅ **Capability names must match exactly** between sheets  
✅ **Application names must match exactly** between sheets  
✅ **Use consistent naming** (case-sensitive)  
✅ **Keep it simple** - clear, full names work best  

❌ **Don't use special characters** in names  
❌ **Don't leave blank rows** in the middle of data  
❌ **Don't use abbreviations** unless necessary  

## For End Users (Web Part Users)

### Finding Information

**To see applications for a capability**:
1. Enter capability name in search box
2. Click "Find Applications"  
3. View table of supporting applications

**To understand relationships**:
- **Primary**: Core applications for this capability
- **Supporting**: Applications that help but aren't essential

### Tips

- **Use exact names** - search is case-sensitive
- **Try partial names** if you're unsure of the full name
- **Check with business users** if data seems outdated

## For Developers

### Key Files

**Excel workbook structure**:
```
Sheet 1: Capabilities (CapabilityName, ParentCapabilityName, Level, Tier, Definition, Owner)
Sheet 2: Applications (ApplicationName, Category, Vendor, Status)  
Sheet 3: CapabilityApplications (CapabilityName, ApplicationName, UsageType, Notes)
```

**Core service**:
```typescript
// Get workbook ID
const workbookId = "YOUR-EXCEL-FILE-ID";

// Read relationships
const response = await graphClient.get(
  `/me/drive/items/${workbookId}/workbook/worksheets/CapabilityApplications/usedRange`
);
```

### Adding Features

**Common extensions**:
- Search by application name (reverse lookup)
- Display capability hierarchy
- Show all data in filterable tables
- Export functionality

**Integration points**:
- Microsoft Graph API (built into SPFx)
- Excel Online (no setup required)
- SharePoint web parts (standard deployment)

## Architecture Benefits

### Why This Works

✅ **Zero maintenance** - Excel Online handles everything  
✅ **Unlimited scale** - no database limits  
✅ **Business user friendly** - everyone knows Excel  
✅ **Real-time updates** - changes appear automatically  
✅ **Disaster recovery** - standard Office 365 backup  
✅ **Version control** - Excel Online tracks changes  
✅ **Collaboration** - multiple users can edit safely  

### What Makes It Different

**Traditional approach**: Complex database → Admin maintenance → Technical users only  
**This approach**: Simple Excel → Zero maintenance → Business users empowered  

## Troubleshooting

### Common Issues

**"Data not loading"**:
- Check workbook ID is correct
- Verify file permissions
- Try refreshing the page

**"Capability not found"**:
- Check spelling and case
- Verify capability exists in Excel
- Look for extra spaces in Excel data

**"Changes not appearing"**:
- Wait 1-2 minutes for sync
- Check Excel file was saved
- Refresh SharePoint page

### Getting Help

**For data issues**: Business users can fix directly in Excel  
**For web part issues**: Contact your SharePoint admin  
**For new features**: Developers can extend easily  

## Best Practices

### Data Management

**Regular cleanup**:
- Remove outdated applications
- Update capability definitions  
- Consolidate duplicate entries

**Quality control**:
- Use data validation in Excel
- Establish naming conventions
- Review changes periodically

### User Training

**Business users need to know**:
- How to edit Excel safely
- Naming consistency rules
- Where to find the workbook

**End users need to know**:
- How to search effectively
- What the relationship types mean
- Who to contact for data updates

## Scale Considerations

**Current capacity**:
- Unlimited capabilities and applications
- Unlimited relationships
- 50+ concurrent users supported

**Growth path**:
- Single workbook handles most organizations
- Can split by business unit if needed
- Multiple workbooks with consolidation views

**Performance expectations**:
- Search results in 1-2 seconds
- Real-time collaboration in Excel
- Automatic updates to web part

This system is designed to grow with your organization while maintaining simplicity and zero maintenance overhead.