# Technical Architecture - API Analytics Integration

## **Technical Architecture**

### **Data Flow Architecture**
```
MyCare API/Frontend → Freshpaint (HIPAA Middleware) → Google Analytics 4 → GA4 Dashboards
```

**Note**: No custom dashboards needed - use GA4's built-in reporting and dashboard capabilities.

### **Existing Data Foundation**
The MyCare API already captures rich analytics data through its audit system:

```javascript
// AuditLog table structure (already exists)
{
  TableName: "Case|Patient|User|Organization|TreatmentOption|Resource",
  AuditAction: "Insert|Update|Read|Delete", 
  UserID: userId,
  TableID: recordId,
  ActivityDate: timestamp,
  Comment: "SP_INSERT_CASE|SP_UPDATE_Patient|etc"
}
```

### **User Context Available**
```javascript
// Rich user context from existing auth system
{
  userId: number,
  userEmail: string,
  roleId: number, // 1-9 (APP, Admin, Nurse, Physician, Patient, etc.)
  organizationId: number,
  language: "en|es",
  userAuthenticated: boolean
}
```

---

## **Implementation Strategy**

### **Phase 1: Audit Log Integration (Week 1-2)**
**Leverage Existing Audit Trail**

```javascript
// Create webhook/function to send audit events to Freshpaint
const auditEvents = await prisma.auditLog.findMany({
  where: { ActivityDate: { gte: lastSync } },
  include: { User: true }
});

// Map to Freshpaint events
auditEvents.forEach(event => {
  freshpaint.track(event.AuditAction, {
    table_name: event.TableName,
    user_id: event.UserID,
    user_role: getUserRole(event.User.RoleID),
    organization_id: event.User.OrganizationID,
    module_id: extractModuleId(event),
    timestamp: event.ActivityDate,
    environment: getEnvironment()
  });
});
```

**Immediate Analytics Available:**
- Case creation patterns by module and organization
- User activity by role and time
- Treatment plan selection trends
- Resource access patterns
- System performance metrics

### **Phase 2: Enhanced Event Tracking (Week 3-4)**
**Add Custom Events to Existing API Endpoints**

```javascript
// Example: Enhanced case creation tracking
module.exports = withAuth(async function (context, req) {
  // Existing logic...
  
  if (success) {
    // Add Freshpaint tracking
    freshpaint.track('case_created', {
      case_id: newCaseId,
      module_id: moduleId,
      user_role: roleId,
      organization_id: userOrgId,
      patient_age_range: ageRangeId,
      has_notes: !!caseNote,
      creation_time_seconds: creationTime,
      data_source: 'manual|import|template'
    });
  }
});
```

### **Phase 3: Clinical Workflow Analytics (Week 5-6)**
**Track Complete Treatment Workflows**

```javascript
// Treatment plan selection tracking
const trackTreatmentPlan = (caseId, treatmentOption, selectedOption) => {
  freshpaint.track('treatment_plan_selected', {
    case_id: caseId,
    treatment_option: treatmentOption,
    selected_option: selectedOption,
    user_role: context.user.roleId,
    module_id: context.user.moduleId,
    organization_id: context.user.organizationId
  });
};

// Resource utilization tracking
const trackResourceAccess = (resourceId, resourceType, accessMethod) => {
  freshpaint.track('resource_accessed', {
    resource_id: resourceId,
    resource_type: resourceType,
    access_method: accessMethod,
    user_role: context.user.roleId,
    module_id: context.user.moduleId
  });
};
```
