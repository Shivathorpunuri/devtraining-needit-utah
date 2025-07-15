# ServiceNow: How to Populate Current Logged-In User Details in Incident Forms

This guide provides multiple methods to automatically populate the current logged-in user details in ServiceNow incident forms, saving users time and improving data accuracy.

## Table of Contents
1. [Default Value Method (Simplest)](#default-value-method-simplest)
2. [Client Script Method (onLoad)](#client-script-method-onload)
3. [Business Rule Method (Server-side)](#business-rule-method-server-side)
4. [Dictionary Entry Method](#dictionary-entry-method)
5. [Common User Object Methods](#common-user-object-methods)
6. [Best Practices](#best-practices)

---

## Default Value Method (Simplest)

The easiest way to populate user details is by setting default values in the field configuration.

### For Catalog Items Variables

Navigate to **Service Catalog → Catalog Definitions → Maintain Items** → select your catalog item → Variables tab

For each variable, set the **Default Value** with the following (must include `javascript:` prefix):

```javascript
// User's full name
javascript: gs.getUser().getDisplayName();

// User's first name
javascript: gs.getUser().getFirstName();

// User's last name  
javascript: gs.getUser().getLastName();

// User's email
javascript: gs.getUser().getEmail();

// User's phone number
javascript: gs.getUser().getRecord().getDisplayValue('phone');

// User's department
javascript: gs.getUser().getRecord().getDisplayValue('department');

// User's company
javascript: gs.getUser().getRecord().getDisplayValue('company');

// User's manager
javascript: gs.getUser().getRecord().getDisplayValue('manager');

// User's location
javascript: gs.getUser().getRecord().getDisplayValue('location');

// User ID
javascript: gs.getUserID();

// User's company ID
javascript: gs.getUser().getCompanyID();
```

### For Dictionary Fields

Navigate to **System Definition → Tables** → select table → select field → Attributes

Add to the **Default Value** field:

```javascript
javascript: gs.getUserID();  // For assigned_to or caller_id fields
javascript: gs.getUser().getCompanyID();  // For company fields
```

---

## Client Script Method (onLoad)

Create a client script that runs when the form loads to populate fields dynamically.

### Basic onLoad Client Script

**Type:** onLoad  
**Table:** incident (or your target table)  
**Script:**

```javascript
function onLoad() {
    // Only run for new records
    if (g_form.isNewRecord()) {
        
        // Populate caller with current user
        g_form.setValue('caller_id', g_user.userID);
        
        // Populate company if empty
        if (g_form.getValue('company') == '') {
            g_form.setValue('company', g_user.companyID);
        }
        
        // Set location based on user's location
        var userLocation = g_user.getRecord().location;
        if (userLocation) {
            g_form.setValue('location', userLocation);
        }
        
        // Auto-populate description with user context
        var currentDescription = g_form.getValue('description');
        if (currentDescription == '') {
            var userInfo = "Submitted by: " + g_user.firstName + " " + g_user.lastName + 
                          " (" + g_user.email + ")\n" +
                          "Department: " + g_user.getRecord().department + "\n\n";
            g_form.setValue('description', userInfo);
        }
    }
}
```

### Advanced Client Script with GlideAjax

For more complex scenarios where you need server-side user data:

**Client Script:**
```javascript
function onLoad() {
    if (g_form.isNewRecord()) {
        // Call server-side script to get user details
        var ga = new GlideAjax('UserDetailsAjax');
        ga.addParam('sysparm_name', 'getUserDetails');
        ga.addParam('sysparm_user_id', g_user.userID);
        ga.getXML(populateUserFields);
    }
    
    function populateUserFields(response) {
        var answer = response.responseXML.documentElement.getAttribute("answer");
        if (answer) {
            var userDetails = JSON.parse(answer);
            g_form.setValue('caller_id', userDetails.user_id);
            g_form.setValue('company', userDetails.company);
            g_form.setValue('u_department', userDetails.department);
            g_form.setValue('u_phone', userDetails.phone);
        }
    }
}
```

**Required Script Include (UserDetailsAjax):**
```javascript
var UserDetailsAjax = Class.create();
UserDetailsAjax.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    
    getUserDetails: function() {
        var userId = this.getParameter('sysparm_user_id');
        var grUser = new GlideRecord('sys_user');
        
        if (grUser.get(userId)) {
            var userDetails = {
                user_id: grUser.sys_id.toString(),
                email: grUser.email.toString(),
                phone: grUser.phone.toString(),
                department: grUser.department.toString(),
                company: grUser.company.toString(),
                location: grUser.location.toString(),
                manager: grUser.manager.toString()
            };
            return JSON.stringify(userDetails);
        }
        return '';
    },
    
    type: 'UserDetailsAjax'
});
```

---

## Business Rule Method (Server-side)

Create a business rule to automatically populate user details when records are created.

**Name:** Auto-populate User Details  
**Table:** incident  
**When:** before  
**Insert:** true  
**Advanced:** true  
**Script:**

```javascript
(function executeRule(current, previous /*null when async*/) {
    
    // Only run for new records
    if (current.isNewRecord()) {
        
        // Get current logged-in user
        var currentUser = gs.getUser();
        var userId = currentUser.getID();
        
        // Populate caller_id if empty
        if (current.caller_id.nil()) {
            current.caller_id = userId;
        }
        
        // Populate company if empty
        if (current.company.nil()) {
            current.company = currentUser.getCompanyID();
        }
        
        // Get additional user details
        var grUser = new GlideRecord('sys_user');
        if (grUser.get(userId)) {
            
            // Populate location if empty
            if (current.location.nil() && !grUser.location.nil()) {
                current.location = grUser.location;
            }
            
            // Add user information to work notes
            var userInfo = "Record created by: " + grUser.getDisplayValue() + 
                          " (" + grUser.email + ")\n" +
                          "Department: " + grUser.department.getDisplayValue() + "\n" +
                          "Phone: " + grUser.phone;
            
            current.work_notes = userInfo;
        }
    }
    
})(current, previous);
```

---

## Dictionary Entry Method

Configure default values directly in the dictionary for specific fields.

### For Assigned To field

1. Navigate to **System Definition → Tables → Incident → Assigned to**
2. In the **Default Value** field, enter:
```javascript
javascript: gs.getUserID();
```

### For Company field

1. Navigate to **System Definition → Tables → Incident → Company**
2. In the **Default Value** field, enter:
```javascript
javascript: gs.getUser().getCompanyID();
```

---

## Common User Object Methods

### Server-side (Business Rules, Script Includes)

```javascript
// Get current user object
var currentUser = gs.getUser();

// Basic user information
var userId = currentUser.getID();
var userName = currentUser.getName();
var userDisplayName = currentUser.getDisplayName();
var firstName = currentUser.getFirstName();
var lastName = currentUser.getLastName();
var email = currentUser.getEmail();
var companyId = currentUser.getCompanyID();

// Check user roles
var hasItilRole = currentUser.hasRole('itil');
var isAdmin = currentUser.hasRole('admin');

// Get user record for additional fields
var grUser = new GlideRecord('sys_user');
if (grUser.get(userId)) {
    var phone = grUser.phone.toString();
    var department = grUser.department.getDisplayValue();
    var location = grUser.location.getDisplayValue();
    var manager = grUser.manager.getDisplayValue();
    var title = grUser.title.toString();
}
```

### Client-side (Client Scripts, UI Policies)

```javascript
// Basic user information (available globally)
var userId = g_user.userID;
var userName = g_user.userName;
var firstName = g_user.firstName;
var lastName = g_user.lastName;
var email = g_user.email;
var companyId = g_user.companyID;

// Check user roles
var hasItilRole = g_user.hasRole('itil');

// Get additional user details (requires GlideAjax or form reference)
var userRecord = g_user.getRecord();
if (userRecord) {
    var department = userRecord.department;
    var phone = userRecord.phone;
    var location = userRecord.location;
}
```

---

## Best Practices

### 1. **Performance Considerations**
- Use client-side methods when possible to reduce server load
- Cache user information in client-side variables if needed multiple times
- Avoid excessive GlideRecord queries in frequently triggered business rules

### 2. **Security & Privacy**
- Be mindful of data privacy when auto-populating sensitive information
- Ensure users have appropriate permissions to view populated data
- Consider using ACLs to control field visibility

### 3. **User Experience**
- Allow users to modify auto-populated fields when necessary
- Provide clear indication when fields are auto-populated
- Don't override user-entered data unnecessarily

### 4. **Error Handling**
```javascript
// Always check if user exists and has required data
function safeGetUserValue(field) {
    try {
        var user = gs.getUser();
        if (user && user.getRecord()) {
            return user.getRecord().getDisplayValue(field) || '';
        }
    } catch (e) {
        gs.log('Error getting user field ' + field + ': ' + e.message);
    }
    return '';
}
```

### 5. **Testing**
- Test with different user roles and permissions
- Verify behavior for guest users or users without complete profiles
- Test in both standard UI and Service Portal

### 6. **Documentation**
- Document which fields are auto-populated and under what conditions
- Provide training for users on how the auto-population works
- Maintain clear business rules about when to override vs. preserve user data

---

## Common Use Cases

### 1. **Self-Service Portal**
Auto-populate incident caller with portal user:
```javascript
// In catalog client script or incident business rule
if (gs.getUser().userID != 'guest') {
    current.caller_id = gs.getUserID();
}
```

### 2. **Manager Assignment**
Auto-assign incidents to user's manager:
```javascript
var grUser = new GlideRecord('sys_user');
if (grUser.get(current.caller_id)) {
    if (!grUser.manager.nil()) {
        current.assigned_to = grUser.manager;
    }
}
```

### 3. **Department-based Routing**
Route based on user's department:
```javascript
var grUser = new GlideRecord('sys_user');
if (grUser.get(current.caller_id)) {
    var dept = grUser.department.getDisplayValue();
    if (dept == 'IT') {
        current.assignment_group = 'IT Support';
    } else if (dept == 'HR') {
        current.assignment_group = 'HR Team';
    }
}
```

This guide provides comprehensive methods for populating user details in ServiceNow incident forms. Choose the method that best fits your specific requirements and organizational needs.