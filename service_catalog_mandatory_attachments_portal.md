# Making Attachments Mandatory on Service Catalogue in Portal (Not Native UI)

## Overview

Making attachments mandatory in ServiceNow Service Portal is significantly more challenging than in the native UI because traditional `g_form` APIs like `getControl()` and `getElement()` don't exist in Service Portal. This guide provides several approaches to implement mandatory attachments specifically for portal environments.

## Key Challenges in Service Portal

1. **No traditional g_form APIs** - `getControl()`, `getElement()`, and `gel()` don't exist
2. **No synchronous server calls** - Can't use synchronous GlideRecord or GlideAjax in onSubmit
3. **Different DOM structure** - Portal uses Angular-based widgets vs traditional forms
4. **Limited attachment validation** - No built-in attachment validation methods

## Solution Approaches

### Approach 1: Catalog Client Script with spForm (Recommended)

This approach uses the `spForm` object available in Service Portal to check attachments before submission.

#### Implementation Steps:

1. **Create a Catalog Client Script**
   - **Type:** onSubmit
   - **UI Type:** Service Portal
   - **Catalog Item:** Specific catalog item or leave blank for all items
   - **Applies to:** Catalog Item

2. **Client Script Code:**

```javascript
function onSubmit() {
    // Check if we're in Service Portal
    if (typeof spForm !== 'undefined') {
        // Get attachments using spForm
        var attachments = spForm.getAttachments();
        
        // Check if at least one attachment is required
        if (!attachments || attachments.length === 0) {
            spForm.addErrorMessage("At least one attachment is required before submission.");
            return false;
        }
        
        // Optional: Check for specific number of attachments
        var requiredCount = 2;
        if (attachments.length < requiredCount) {
            spForm.addErrorMessage("At least " + requiredCount + " attachments are required.");
            return false;
        }
        
        // Optional: Check for specific file types
        var allowedTypes = ['pdf', 'doc', 'docx', 'jpg', 'png'];
        var hasValidType = false;
        
        for (var i = 0; i < attachments.length; i++) {
            var fileName = attachments[i].name || '';
            var fileExtension = fileName.split('.').pop().toLowerCase();
            
            if (allowedTypes.indexOf(fileExtension) !== -1) {
                hasValidType = true;
                break;
            }
        }
        
        if (!hasValidType) {
            spForm.addErrorMessage("Please upload files with allowed formats: " + allowedTypes.join(', '));
            return false;
        }
    }
    
    return true;
}
```

### Approach 2: Widget Customization

For more advanced control, you can clone the catalog form widget and add custom validation.

#### Steps:

1. **Clone the Catalog Form Widget**
   - Go to Service Portal > Widgets
   - Find "SC Catalog Item" widget
   - Clone it with a new name

2. **Modify Client Controller:**

```javascript
// In the widget's client controller
function($scope, spUtil) {
    var c = this;
    
    // Override submit function
    c.submitCatalogItem = function() {
        // Check attachments before submission
        var attachments = c.data.attachments || [];
        
        if (attachments.length === 0) {
            spUtil.addErrorMessage("Please attach at least one file before submitting.");
            return;
        }
        
        // Proceed with original submission logic
        c.originalSubmit();
    };
}
```

3. **Update Catalog Item to use custom widget**

### Approach 3: Enhanced Solution using SN Pro Tips Utility

The SN Pro Tips team has created a comprehensive solution that adds missing functionality to Service Portal.

#### Features:
- `spForm.getAttachments()` method
- `spForm.getElement()` and `spForm.getControl()` methods
- Variable name and sys_id access
- Better attachment validation capabilities

#### Implementation:

1. **Download and install the SN Pro Tips Service Portal utility**
   - Visit: https://snprotips.com/service-portal-dom-access-mandatory-attachments

2. **Use enhanced client script:**

```javascript
function onSubmit() {
    if (typeof spForm !== 'undefined') {
        // Enhanced attachment checking with specific requirements
        var attachments = spForm.getAttachments();
        
        // Require specific number of attachments
        var requiredCount = 1;
        if (attachments.length < requiredCount) {
            spForm.addErrorMessage("Please attach at least " + requiredCount + " file(s).");
            return false;
        }
        
        // Check for specific attachment types
        var requiredTypes = ['application/pdf']; // MIME types
        var hasRequiredType = attachments.some(function(att) {
            return requiredTypes.indexOf(att.content_type) !== -1;
        });
        
        if (!hasRequiredType) {
            spForm.addErrorMessage("Please attach at least one PDF file.");
            return false;
        }
        
        // Check file size limits
        var maxSizeBytes = 5 * 1024 * 1024; // 5MB
        var oversizedFiles = attachments.filter(function(att) {
            return att.size_bytes > maxSizeBytes;
        });
        
        if (oversizedFiles.length > 0) {
            spForm.addErrorMessage("File size must not exceed 5MB.");
            return false;
        }
    }
    
    return true;
}
```

### Approach 4: UI Policy Alternative

For simpler requirements, you can use UI Policies with conditions.

#### Steps:

1. **Create a UI Policy**
   - **Table:** Requested Item [sc_req_item]
   - **Conditions:** Set based on your requirements
   - **Applies to:** Catalog Item

2. **Add UI Policy Action:**
   - **Type:** Make fields mandatory
   - **Field:** Choose an existing field to make mandatory when no attachments

3. **Combine with Client Script for attachment validation**

## Portal-Specific Considerations

### Conditional Logic for Portal vs Native UI

```javascript
function onSubmit() {
    // Detect if running in Service Portal
    if (typeof parent.angular !== 'undefined' || typeof spForm !== 'undefined') {
        // Portal-specific logic
        return validatePortalAttachments();
    } else {
        // Native UI logic
        return validateNativeAttachments();
    }
}

function validatePortalAttachments() {
    if (typeof spForm !== 'undefined') {
        var attachments = spForm.getAttachments();
        if (!attachments || attachments.length === 0) {
            spForm.addErrorMessage("Attachment required in portal.");
            return false;
        }
    }
    return true;
}

function validateNativeAttachments() {
    var count = getCurrentAttachmentNumber();
    if (count === 0) {
        g_form.addErrorMessage("Attachment required in native UI.");
        return false;
    }
    return true;
}
```

### Testing Different Scenarios

```javascript
function onSubmit() {
    // Multiple detection methods for robustness
    var isPortal = false;
    
    // Method 1: Check for spForm object
    if (typeof spForm !== 'undefined') {
        isPortal = true;
    }
    
    // Method 2: Check for Angular
    if (typeof parent.angular !== 'undefined') {
        isPortal = true;
    }
    
    // Method 3: Check window object
    if (window.parent && window.parent.angular) {
        isPortal = true;
    }
    
    if (isPortal) {
        return validatePortalAttachments();
    } else {
        return validateNativeAttachments();
    }
}
```

## Best Practices

1. **User Experience:** Provide clear error messages about attachment requirements
2. **File Validation:** Check file types, sizes, and naming conventions
3. **Progressive Enhancement:** Ensure functionality works across different portal versions
4. **Testing:** Test thoroughly in both portal and native UI environments
5. **Performance:** Avoid complex file processing that might slow submission
6. **Accessibility:** Ensure attachment requirements are clearly communicated

## Common Issues and Solutions

### Issue 1: spForm undefined
**Solution:** Always check if spForm exists before using it

```javascript
if (typeof spForm !== 'undefined' && spForm.getAttachments) {
    // Safe to use spForm
}
```

### Issue 2: Attachments not detected
**Solution:** Ensure script runs after attachments are fully loaded

```javascript
// Add delay if necessary
setTimeout(function() {
    validateAttachments();
}, 100);
```

### Issue 3: Different behavior in different browsers
**Solution:** Use feature detection instead of browser detection

```javascript
// Feature detection
if ('getAttachments' in spForm) {
    // Use getAttachments
} else {
    // Fallback method
}
```

## Advanced Implementation Example

```javascript
function onSubmit() {
    // Enhanced attachment validation for Service Portal
    if (typeof spForm !== 'undefined') {
        var config = {
            minAttachments: 1,
            maxAttachments: 5,
            allowedTypes: ['pdf', 'doc', 'docx', 'jpg', 'jpeg', 'png'],
            maxSizeMB: 10,
            requiredNaming: false, // Set to true if specific naming required
            namePattern: /^[A-Za-z0-9_-]+\.(pdf|doc|docx)$/i
        };
        
        return validateAttachmentsAdvanced(config);
    }
    
    return true;
}

function validateAttachmentsAdvanced(config) {
    var attachments = spForm.getAttachments();
    
    // Check minimum attachments
    if (attachments.length < config.minAttachments) {
        spForm.addErrorMessage("Please attach at least " + config.minAttachments + " file(s).");
        return false;
    }
    
    // Check maximum attachments
    if (attachments.length > config.maxAttachments) {
        spForm.addErrorMessage("Maximum " + config.maxAttachments + " attachments allowed.");
        return false;
    }
    
    // Validate each attachment
    for (var i = 0; i < attachments.length; i++) {
        var attachment = attachments[i];
        var fileName = attachment.name || '';
        var fileExtension = fileName.split('.').pop().toLowerCase();
        var fileSizeMB = (attachment.size_bytes || 0) / (1024 * 1024);
        
        // Check file type
        if (config.allowedTypes.indexOf(fileExtension) === -1) {
            spForm.addErrorMessage("File '" + fileName + "' has an invalid type. Allowed: " + config.allowedTypes.join(', '));
            return false;
        }
        
        // Check file size
        if (fileSizeMB > config.maxSizeMB) {
            spForm.addErrorMessage("File '" + fileName + "' exceeds size limit of " + config.maxSizeMB + "MB.");
            return false;
        }
        
        // Check naming pattern (if required)
        if (config.requiredNaming && !config.namePattern.test(fileName)) {
            spForm.addErrorMessage("File '" + fileName + "' does not meet naming requirements.");
            return false;
        }
    }
    
    return true;
}
```

## Conclusion

Making attachments mandatory in ServiceNow Service Portal requires different approaches than the native UI. The most effective solutions involve:

1. Using Service Portal-specific APIs like `spForm.getAttachments()`
2. Implementing proper portal detection logic
3. Providing clear user feedback
4. Considering third-party utilities for enhanced functionality

Choose the approach that best fits your requirements, technical constraints, and maintenance capabilities. Always test thoroughly across different browsers and portal configurations.