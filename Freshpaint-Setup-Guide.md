# Freshpaint Setup Guide for MyCare API Analytics Integration

**Document Version:** 1.0  
**Date:** January 2025  
**Owner:** Thomas Teruel (Product Manager)  
**Based on:** [Freshpaint Documentation](https://documentation.freshpaint.io/) and API Analytics Integration PRD

---

## **🚀 Freshpaint Setup Process**

### **Phase 1: Account Setup & HIPAA Configuration**

#### **1. Create Freshpaint Account**
- Go to [Freshpaint.io](https://freshpaint.io) and create your master account
- **Important**: One team member creates the master account, then invites others
- Set up your organization profile with healthcare/medical industry details

#### **2. Enable HIPAA Mode**
Based on the [Freshpaint HIPAA documentation](https://documentation.freshpaint.io/), you'll need to:
- **Execute BAA (Business Associate Agreement)** with Freshpaint
- **Enable HIPAA Mode** in your account settings
- This ensures PHI protection and compliance with healthcare regulations

#### **3. Environment Setup**
Create separate environments for your multi-tenant architecture:
- **Production Environment**: For live user data
- **Staging Environment**: For testing and validation  
- **Demo Environment**: For client demonstrations

---

## **Phase 2: Install Freshpaint SDK**

### **Frontend Installation (React App)**
Based on your `mycare-frontend-v3` React application:

```bash
# Install Freshpaint SDK
npm install @freshpaint/freshpaint-js
```

```javascript
// In your main App.jsx or index.jsx
import freshpaint from '@freshpaint/freshpaint-js';

// Environment-aware configuration
const environmentConfig = {
  production: {
    freshpaintToken: 'YOUR_PROD_TOKEN',
    environment: 'production'
  },
  staging: {
    freshpaintToken: 'YOUR_STAGING_TOKEN', 
    environment: 'staging'
  },
  demo: {
    freshpaintToken: 'YOUR_DEMO_TOKEN',
    environment: 'demo'
  }
};

// Auto-detect environment
const envKey = window.location.hostname.includes('staging') ? 'staging' :
               window.location.hostname.includes('demo') ? 'demo' : 'production';

const config = environmentConfig[envKey];

// Initialize Freshpaint
freshpaint.init(config.freshpaintToken, {
  safe_mode: true,
  auto_capture: false, // We'll use custom events from audit logs
  respect_dnt: true,
  secure: true
});
```

### **Backend Integration (API)**
For your `mycare-api-v3` Azure Functions:

```bash
# Install Freshpaint server-side SDK
npm install @freshpaint/freshpaint-node
```

```javascript
// In your API functions
const freshpaint = require('@freshpaint/freshpaint-node');

// Initialize with your server-side token
freshpaint.init('YOUR_SERVER_TOKEN');

// Example: Audit log webhook function
module.exports = async function (context, req) {
  const auditEvents = await prisma.auditLog.findMany({
    where: { 
      ActivityDate: { gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
    },
    include: { 
      User: {
        include: {
          Organization: true,
          Role: true
        }
      }
    }
  });

  for (const event of auditEvents) {
    await freshpaint.track(event.AuditAction, {
      table_name: event.TableName,
      user_id: event.UserID,
      user_role: event.User.Role.RoleName,
      organization_id: event.User.OrganizationID,
      organization_name: event.User.Organization.OrgName,
      timestamp: event.ActivityDate,
      environment: getEnvironment()
    });
  }
};
```

---

## **Phase 3: Configure Destinations**

### **Google Analytics 4 Setup**
Based on the [Freshpaint GA4 documentation](https://documentation.freshpaint.io/):

#### **1. Create GA4 Properties**
- **Production**: `G-XXXXXXXXXX`
- **Staging**: `G-YYYYYYYYYY` 
- **Demo**: `G-ZZZZZZZZZZ`

#### **2. Configure Freshpaint Destinations**
- Go to Freshpaint Admin Panel → Destinations
- Add Google Analytics 4 Server-Side destination
- Configure each environment to point to correct GA4 property
- Enable HIPAA-compliant data sanitization

### **Environment Separation**
Following the [Environment Separation Strategy](https://documentation.freshpaint.io/):

```javascript
// Ensure environment tagging on every event
function trackEvent(eventName, properties = {}) {
  const environment = getEnvironment();
  freshpaint.track(eventName, {
    ...properties,
    environment,
    // Add other global properties
    app_version: process.env.APP_VERSION,
    deployment_date: process.env.DEPLOYMENT_DATE
  });
}
```

---

## **Phase 4: Event Configuration**

### **Leverage Your 247 Stored Procedures**
Based on your API analytics integration, you can immediately start tracking:

```javascript
// Map your existing audit events to Freshpaint
const auditEventMapping = {
  'SP_INSERT_CASE': 'case_created',
  'SP_UPDATE_CASE': 'case_updated', 
  'SP_INSERT_TreatmentOption': 'treatment_option_created',
  'SP_ActivateCaseResource': 'case_resource_activated',
  'SP_INSERT_Patient': 'patient_created',
  'SP_UPDATE_Patient': 'patient_updated',
  'SP_INSERT_User': 'user_created',
  'SP_UPDATE_User': 'user_updated',
  'SP_INSERT_Organization': 'organization_created',
  'SP_UPDATE_Organization': 'organization_updated',
  // ... all 247 procedures
};

// Track events with rich context
auditEvents.forEach(event => {
  freshpaint.track(auditEventMapping[event.Comment], {
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

### **Enhanced Event Tracking**
Add custom events to existing API endpoints:

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

---

## **Phase 5: Testing & Validation**

### **Use Freshpaint's Testing Tools**
- **Live View**: Real-time event monitoring
- **Event Verification**: Validate event structure and data
- **Debug Mode**: Test events in staging environment

### **Compliance Validation**
- Verify no PHI is being sent to GA4
- Test environment separation
- Validate HIPAA compliance with audit logs

### **Testing Checklist**
- [ ] Events flowing to correct GA4 properties
- [ ] Environment separation working
- [ ] No PHI exposure in analytics events
- [ ] Audit log integration operational
- [ ] Custom event tracking functional

---

## **Phase 6: Go Live**

### **Production Deployment**
1. **Deploy to staging** first for validation
2. **Test with real audit data** for 48+ hours
3. **Validate GA4 reports** show correct data
4. **Deploy to production** with monitoring

### **Monitoring Setup**
- Set up Freshpaint destination monitoring
- Configure alerts for failed events
- Monitor GA4 data quality

---

## **🎯 Key Advantages for Your Implementation**

### **Immediate Value**
- **247 events ready to track** from your existing audit system
- **No custom dashboard development** needed - use GA4's built-in reports
- **HIPAA compliance built-in** with Freshpaint's healthcare features

### **Implementation Timeline**
- **Week 1**: Account setup, BAA execution, SDK installation
- **Week 2**: Environment configuration, audit log integration
- **Week 3-4**: Custom event tracking, testing
- **Week 5-6**: GA4 reports, stakeholder training
- **Week 7-8**: Optimization, production monitoring

---

## **📋 Implementation Checklist**

### **Phase 1: Foundation (Weeks 1-2)**
- [ ] Create Freshpaint account
- [ ] Execute BAA with Freshpaint
- [ ] Enable HIPAA Mode
- [ ] Set up environments (prod/staging/demo)
- [ ] Install Freshpaint SDK in frontend
- [ ] Install Freshpaint SDK in backend
- [ ] Configure environment detection

### **Phase 2: Integration (Weeks 3-4)**
- [ ] Create GA4 properties for each environment
- [ ] Configure Freshpaint destinations
- [ ] Implement audit log webhook integration
- [ ] Map 247 stored procedures to events
- [ ] Add custom event tracking to API endpoints
- [ ] Test environment separation

### **Phase 3: Validation (Weeks 5-6)**
- [ ] Use Freshpaint Live View for testing
- [ ] Validate event structure and data
- [ ] Test HIPAA compliance
- [ ] Verify no PHI exposure
- [ ] Set up GA4 reports and dashboards
- [ ] Configure monitoring and alerts

### **Phase 4: Production (Weeks 7-8)**
- [ ] Deploy to staging environment
- [ ] Run 48+ hour validation test
- [ ] Deploy to production
- [ ] Train stakeholders on GA4 interface
- [ ] Set up production monitoring
- [ ] Document procedures and maintenance

---

## **🔧 Technical Configuration Examples**

### **Environment Configuration**
```javascript
// Environment-aware configuration
const environmentConfig = {
  production: {
    freshpaintToken: 'YOUR_PROD_TOKEN',
    ga4MeasurementId: 'G-XXXXXXXXXX',
    environment: 'production'
  },
  staging: {
    freshpaintToken: 'YOUR_STAGING_TOKEN',
    ga4MeasurementId: 'G-YYYYYYYYYY',
    environment: 'staging'
  },
  demo: {
    freshpaintToken: 'YOUR_DEMO_TOKEN',
    ga4MeasurementId: 'G-ZZZZZZZZZZ',
    environment: 'demo'
  }
};
```

### **User Role Mapping**
```javascript
// User roles from your authConstants.js
const userRoleMapping = {
  1: 'APP',
  2: 'MCG_Administrator',
  3: 'Auditor',
  4: 'Nurse',
  5: 'Administrator',
  6: 'Physician',
  7: 'Scribe',
  8: 'Patient',
  9: 'CareGiver'
};
```

### **Clinical Module Mapping**
```javascript
// Clinical modules available for tracking
const clinicalModules = {
  'liver': 'Liver Cancer Module',
  'lung': 'Lung Cancer Module',
  'brain': 'Brain Cancer Module',
  'prostate': 'Prostate Cancer Module',
  'breast': 'Breast Cancer Module'
};
```

---

## **📚 Additional Resources**

### **Freshpaint Documentation**
- [Freshpaint Documentation](https://documentation.freshpaint.io/)
- [HIPAA Mode](https://documentation.freshpaint.io/)
- [Google Analytics 4 Integration](https://documentation.freshpaint.io/)
- [Environment Separation Strategy](https://documentation.freshpaint.io/)

### **Related Documents**
- `01-Executive-Summary.md` - Project overview and goals
- `02-Technical-Architecture.md` - Implementation strategy
- `04-Implementation-Roadmap.md` - Detailed timeline
- `07-Complete-Event-List.md` - All 247 trackable events

---

## **🚨 Important Notes**

### **HIPAA Compliance**
- **Never send PHI** to GA4 or any analytics platform
- **Use Freshpaint's HIPAA Mode** for healthcare compliance
- **Leverage existing audit sanitization** from your API
- **Test compliance** before production deployment

### **Environment Separation**
- **Separate GA4 properties** for each environment
- **Test environment separation** thoroughly
- **Monitor for data contamination** between environments
- **Use environment tags** on all events

### **Performance Considerations**
- **Async processing** for audit log integration
- **Batch operations** for high-volume events
- **Monitor performance impact** on API response times
- **Set up alerts** for failed event tracking

---

**Next Steps:**
1. **Create Freshpaint account** and execute BAA
2. **Install SDKs** in both frontend and backend
3. **Configure GA4 destinations** for each environment
4. **Start with audit log integration** (Phase 1 of your roadmap)
5. **Use Freshpaint's testing tools** to validate implementation

The beauty of your approach is that you're leveraging existing audit data rather than starting from scratch, which means you'll have rich analytics insights immediately after the basic setup is complete!
