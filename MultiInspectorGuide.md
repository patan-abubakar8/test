# Multi-Inspector Implementation Guide

This guide provides a detailed step-by-step approach to implementing multi-inspector support for sub-assessments. Follow these steps to replicate the functionality on your development environment.

## Overview
The goal is to allow a sub-assessment to be assigned to multiple inspectors simultaneously, while maintaining a "Max Inspector Count" limit defined at the parent assessment level.

---

## Step 1: Database Model Changes

### 1.1 Update `Assessment` Model
Add a property to track the allowed number of inspectors.
**File:** `MIAudit.DataContext\Models\Assessment.cs`
```csharp
public partial class Assessment
{
    // ... existing properties ...
    public int MaxInspectorCount { get; set; } // Add this line
}
```

### 1.2 Create `SubAssessmentInspector` Model [NEW]
This join table handles the many-to-many relationship between Sub-Assessments and Users.
**File:** `MIAudit.DataContext\Models\SubAssessmentInspector.cs`
```csharp
using System;
using System.Collections.Generic;

namespace MIAudit.DataContext.Models
{
    public partial class SubAssessmentInspector
    {
        public int Id { get; set; }
        public int SubAssessmentId { get; set; }
        public int InspectorId { get; set; }

        public virtual SubAssessment SubAssessment { get; set; }
        public virtual User Inspector { get; set; }
    }
}
```

---

## Step 2: Database Context Configuration

### 2.1 Register `DbSet`
**File:** `MIAudit.DataContext\Models\MIAuditDBContext.cs`
```csharp
public virtual DbSet<SubAssessmentInspector> SubAssessmentInspectors { get; set; }
```

### 2.2 Configure Relationships in `OnModelCreating`
**File:** `MIAudit.DataContext\Models\MIAuditDBContext.cs`
```csharp
modelBuilder.Entity<SubAssessmentInspector>(entity =>
{
    entity.ToTable("SubAssessmentInspector");

    entity.HasOne(d => d.SubAssessment)
        .WithMany(p => p.SubAssessmentInspectors)
        .HasForeignKey(d => d.SubAssessmentId)
        .HasConstraintName("FK_SubAssessmentInspector_SubAssessment");

    entity.HasOne(d => d.Inspector)
        .WithMany(p => p.SubAssessmentInspectors)
        .HasForeignKey(d => d.InspectorId)
        .OnDelete(DeleteBehavior.ClientSetNull)
        .HasConstraintName("FK_SubAssessmentInspector_User");
});
```

---

## Step 3: Infrastructure (Unit of Work)

### 3.1 Update `IUnitOfWork`
**File:** `MIAudit.Services\Infastructure\Contract\UnitofworkInterface\IUnitOfWork.cs`
```csharp
IBaseRepository<SubAssessmentInspector> repSubAssessmentInspectors { get; }
```

### 3.2 Update `UnitOfWork`
**File:** `MIAudit.Services\Infastructure\Contract\UnitofworkInterface\UnitOfWork.cs`
```csharp
// Define member
BaseRepository<SubAssessmentInspector> _subAssessmentInspector = null;

// Implement property
public IBaseRepository<SubAssessmentInspector> repSubAssessmentInspectors => 
    _subAssessmentInspector ?? (_subAssessmentInspector = new BaseRepository<SubAssessmentInspector>(_httpContextAccessor, Configuration));
```

---

## Step 4: ViewModel Updates

### 4.1 Update `AssessmentViewModel`
**File:** `MIAudit.Common\ViewModel\AssessmentViewModel.cs`
```csharp
public int MaxInspectorCount { get; set; }
```

### 4.2 Update `SubAssessmentViewModel`
**File:** `MIAudit.Common\ViewModel\SubAssessmentViewModel.cs`
```csharp
public int[] InspectorIds { get; set; } // For capturing multiple IDs from dropdown
public string[] InspectorNames { get; set; } // For display
public int MaxInspectorCount { get; set; } // For UI validation
```

---

## Step 5: Business Logic (`AssessmentService`)

### 5.1 Update `GetAssessmentById`
Ensure `MaxInspectorCount` is retrieved.
```csharp
MaxInspectorCount = u.MaxInspectorCount
```

### 5.2 Update `SaveSubAssessments`
Handle multiple inspectors when creating a sub-assessment.
```csharp
if (model.InspectorIds != null && model.InspectorIds.Any())
{
    // Link to Join Table
    foreach (var inspectorId in model.InspectorIds)
    {
        var sai = new SubAssessmentInspector
        {
            SubAssessmentId = subassessmentData.Id,
            InspectorId = inspectorId
        };
        _unitOfWork.repSubAssessmentInspectors.Insert(sai);
    }
}
```

### 5.3 Update `UpdateSubAssessment`
Refresh inspector associations.
```csharp
// Remove existing
var existingInspectors = _unitOfWork.repSubAssessmentInspectors.GetAll(c => c.SubAssessmentId == model.Id).ToList();
_unitOfWork.repSubAssessmentInspectors.RemoveRange(existingInspectors);

// Add new
if (model.InspectorIds != null)
{
    foreach (var inspectorId in model.InspectorIds)
    {
        var sai = new SubAssessmentInspector { SubAssessmentId = model.Id, InspectorId = inspectorId };
        _unitOfWork.repSubAssessmentInspectors.Insert(sai);
    }
}
```

---

## Step 6: Controller Logic (`AssessmentController`)

### 6.1 Update `_AddSubAssessment`
Prepare the inspector list and filter out those already assigned to other sub-assessments.
```csharp
// Get inspectors assigned to OTHER sub-assessments of the same parent
var assignedToOthers = subAssessModel
    .Where(s => s.Id != addId)
    .SelectMany(s => s.InspectorIds ?? new int[0])
    .Distinct()
    .ToList();

allInspetors = allInspetors.Where(d => !assignedToOthers.Contains(d.Id)).ToList();
ViewBag.InspectorDropdown = allInspetors;
```

---

## Step 7: UI Changes (Views)

### 7.1 Update `_AddSubAssessment.cshtml`
Change the single select to a multiple select dropdown with limits.
```html
<select id="insID" class="form-control" multiple="multiple">
    @foreach (var item in (IEnumerable<MIAudit.Common.ViewModel.UserViewModel>)ViewBag.InspectorDropdown)
    {
        <option value="@item.Id">@item.Name</option>
    }
</select>

<script>
    // Initialize with limit
    var maxInspectors = @ViewBag.MaxInspectorCount;
    $('#insID').on('change', function() {
        if($(this).val().length > maxInspectors) {
            alert("Maximum " + maxInspectors + " inspectors allowed.");
            // Logic to revert selection
        }
    });
</script>
```

### 7.2 Update `_GetAllSubAssessments.cshtml`
Display the list of comma-separated names.
```html
<td>@string.Join(", ", item.InspectorNames)</td>
```
