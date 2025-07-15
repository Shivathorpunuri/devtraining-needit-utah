# Native ServiceNow Service Portal - Making Attachments Mandatory

## The Problem with spForm
`spForm` is **NOT** a native ServiceNow object. It's from third-party utilities. Here are the **native solutions** that work with standard ServiceNow Service Portal.

## Solution 1: DOM-Based Attachment Detection (Native)

### Catalog Client Script (Native Approach)
```javascript
function onSubmit() {
    // Check if we're in Service Portal
    if (typeof parent.angular !== 'undefined') {
        
        // Method 1: Check attachment widget directly
        var attachmentElements = document.querySelectorAll('.attachment-card, .file-attachment, [data-file-name]');
        
        if (attachmentElements.length === 0) {
            g_form.addErrorMessage("At least one attachment is required.");
            return false;
        }
        
        // Method 2: Check for attachment container
        var attachmentContainer = document.querySelector('.attachments-container, .file-upload-container');
        if (attachmentContainer) {
            var attachedFiles = attachmentContainer.querySelectorAll('.attachment-item, .uploaded-file');
            if (attachedFiles.length === 0) {
                g_form.addErrorMessage("Please attach at least one file.");
                return false;
            }
        }
    }
    
    return true;
}
```

## Solution 2: Server-Side Validation (Most Reliable)

### Catalog Client Script with GlideAjax
```javascript
function onSubmit() {
    // Only run in Service Portal
    if (typeof parent.angular !== 'undefined') {
        
        // Use GlideAjax to check attachments on server
        var ga = new GlideAjax('AttachmentValidator');
        ga.addParam('sysparm_name', 'checkCatalogAttachments');
        ga.addParam('sysparm_table', 'sc_req_item');
        ga.addParam('sysparm_sys_id', g_form.getUniqueValue());
        ga.getXMLWait();
        
        var result = ga.getAnswer();
        
        if (result === 'false') {
            g_form.addErrorMessage("At least one attachment is required before submitting.");
            return false;
        }
    }
    
    return true;
}
```

### Script Include (AttachmentValidator)
```javascript
var AttachmentValidator = Class.create();
AttachmentValidator.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    
    checkCatalogAttachments: function() {
        var tableName = this.getParameter('sysparm_table');
        var sysId = this.getParameter('sysparm_sys_id');
        
        if (!tableName || !sysId) {
            return 'false';
        }
        
        // Check for attachments on the record
        var attachment = new GlideRecord('sys_attachment');
        attachment.addQuery('table_name', tableName);
        attachment.addQuery('table_sys_id', sysId);
        attachment.query();
        
        if (attachment.hasNext()) {
            return 'true';
        }
        
        return 'false';
    },
    
    type: 'AttachmentValidator'
});
```

## Solution 3: Business Rule Validation (Prevention)

### Business Rule on Request Item
```javascript
// Business Rule on sc_req_item
// When: Before Insert/Update
// Condition: current.state == 'submitted_state_value'

(function executeRule(current, previous /*null when async*/) {
    
    // Only validate for portal submissions
    if (gs.getSession().getClientType() === 'service_portal') {
        
        var attachmentCount = new GlideRecord('sys_attachment');
        attachmentCount.addQuery('table_name', 'sc_req_item');
        attachmentCount.addQuery('table_sys_id', current.sys_id);
        attachmentCount.query();
        
        if (attachmentCount.getRowCount() === 0) {
            gs.addErrorMessage("At least one attachment is required for this request.");
            current.setAbortAction(true);
        }
    }
    
})(current, previous);
```

## Solution 4: Widget-Based Detection

### Custom Catalog Form Widget (Clone existing)

#### Client Controller
```javascript
function($scope, $timeout, spUtil) {
    var c = this;
    
    // Override the submit function
    c.submitForm = function() {
        
        // Check for uploaded files
        var uploadedFiles = [];
        
        // Method 1: Check scope for attachments
        if ($scope.attachment && $scope.attachment.files) {
            uploadedFiles = $scope.attachment.files;
        }
        
        // Method 2: Check data object
        if (c.data && c.data.attachments) {
            uploadedFiles = c.data.attachments;
        }
        
        // Method 3: DOM check as fallback
        if (uploadedFiles.length === 0) {
            var fileInputs = document.querySelectorAll('input[type="file"]');
            for (var i = 0; i < fileInputs.length; i++) {
                if (fileInputs[i].files && fileInputs[i].files.length > 0) {
                    uploadedFiles = Array.from(fileInputs[i].files);
                    break;
                }
            }
        }
        
        if (uploadedFiles.length === 0) {
            spUtil.addErrorMessage("Please attach at least one file before submitting.");
            return;
        }
        
        // Proceed with original submission
        c.originalSubmit();
    };
}
```

## Solution 5: Advanced DOM Detection

### Enhanced Client Script with Multiple Detection Methods
```javascript
function onSubmit() {
    // Only run in Service Portal
    if (typeof parent.angular !== 'undefined') {
        
        var hasAttachments = false;
        
        // Method 1: Check for file input elements with files
        var fileInputs = document.querySelectorAll('input[type="file"]');
        for (var i = 0; i < fileInputs.length; i++) {
            if (fileInputs[i].files && fileInputs[i].files.length > 0) {
                hasAttachments = true;
                break;
            }
        }
        
        // Method 2: Check for attachment display elements
        if (!hasAttachments) {
            var attachmentSelectors = [
                '.attachment-card',
                '.file-attachment',
                '.uploaded-file',
                '.attachment-item',
                '[data-file-name]',
                '.file-list-item'
            ];
            
            for (var j = 0; j < attachmentSelectors.length; j++) {
                var elements = document.querySelectorAll(attachmentSelectors[j]);
                if (elements.length > 0) {
                    hasAttachments = true;
                    break;
                }
            }
        }
        
        // Method 3: Check for specific portal attachment text/indicators
        if (!hasAttachments) {
            var pageText = document.body.textContent || document.body.innerText;
            if (pageText.indexOf('uploaded') !== -1 || pageText.indexOf('attached') !== -1) {
                // Additional verification needed here
                var attachmentTexts = document.querySelectorAll('*');
                for (var k = 0; k < attachmentTexts.length; k++) {
                    var text = attachmentTexts[k].textContent;
                    if (text && (text.indexOf('.pdf') !== -1 || text.indexOf('.doc') !== -1 || 
                                text.indexOf('.jpg') !== -1 || text.indexOf('.png') !== -1)) {
                        hasAttachments = true;
                        break;
                    }
                }
            }
        }
        
        if (!hasAttachments) {
            g_form.addErrorMessage("At least one attachment is required before submission.");
            return false;
        }
    }
    
    return true;
}
```

## Solution 6: UI Action with Portal Detection

### UI Action for Catalog Items
```javascript
// UI Action - Client Script
// Condition: gs.action.getGlideURI().toString().indexOf('portal') != -1

function onClick() {
    // Portal-specific validation
    if (typeof parent.angular !== 'undefined') {
        
        // Check using multiple methods
        var attachmentFound = checkForAttachments();
        
        if (!attachmentFound) {
            alert("Please attach at least one file before submitting.");
            return false;
        }
    }
    
    // Continue with submission
    return true;
}

function checkForAttachments() {
    // Check DOM for attachment indicators
    var selectors = [
        'input[type="file"][value!=""]',
        '.attachment-card',
        '.file-attachment',
        '.uploaded-file'
    ];
    
    for (var i = 0; i < selectors.length; i++) {
        var elements = document.querySelectorAll(selectors[i]);
        if (elements.length > 0) {
            return true;
        }
    }
    
    return false;
}
```

## Solution 7: Angular Scope Access (Advanced)

### Accessing Portal's Angular Scope
```javascript
function onSubmit() {
    if (typeof parent.angular !== 'undefined') {
        
        // Try to access Angular scope
        try {
            var element = document.querySelector('[ng-controller]');
            if (element) {
                var scope = parent.angular.element(element).scope();
                
                // Check scope for attachment data
                if (scope && scope.data && scope.data.attachments) {
                    if (scope.data.attachments.length === 0) {
                        g_form.addErrorMessage("At least one attachment is required.");
                        return false;
                    }
                }
            }
        } catch (e) {
            // Fallback to DOM method if Angular access fails
            return checkAttachmentsDOM();
        }
    }
    
    return true;
}

function checkAttachmentsDOM() {
    var fileInputs = document.querySelectorAll('input[type="file"]');
    for (var i = 0; i < fileInputs.length; i++) {
        if (fileInputs[i].files && fileInputs[i].files.length > 0) {
            return true;
        }
    }
    
    g_form.addErrorMessage("At least one attachment is required.");
    return false;
}
```

## Recommended Implementation Strategy

### Step 1: Use Server-Side Validation (Most Reliable)
```javascript
// Catalog Client Script
function onSubmit() {
    if (typeof parent.angular !== 'undefined') {
        var ga = new GlideAjax('AttachmentValidator');
        ga.addParam('sysparm_name', 'validateAttachments');
        ga.addParam('sysparm_req_item_id', g_form.getUniqueValue());
        ga.getXMLWait();
        
        if (ga.getAnswer() === 'false') {
            g_form.addErrorMessage("Please attach at least one file.");
            return false;
        }
    }
    return true;
}
```

### Step 2: Enhanced Script Include
```javascript
var AttachmentValidator = Class.create();
AttachmentValidator.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    
    validateAttachments: function() {
        var reqItemId = this.getParameter('sysparm_req_item_id');
        
        if (!reqItemId) {
            return 'false';
        }
        
        // Check sys_attachment table
        var attachment = new GlideRecord('sys_attachment');
        attachment.addQuery('table_name', 'sc_req_item');
        attachment.addQuery('table_sys_id', reqItemId);
        attachment.query();
        
        return attachment.hasNext().toString();
    },
    
    type: 'AttachmentValidator'
});
```

## Testing Your Implementation

### Test Script for Browser Console
```javascript
// Run in browser console to test attachment detection
function testAttachmentDetection() {
    console.log('=== Attachment Detection Test ===');
    
    // Test 1: File inputs
    var fileInputs = document.querySelectorAll('input[type="file"]');
    console.log('File inputs found:', fileInputs.length);
    
    for (var i = 0; i < fileInputs.length; i++) {
        console.log('Input', i, 'files:', fileInputs[i].files ? fileInputs[i].files.length : 0);
    }
    
    // Test 2: Attachment elements
    var attachmentSelectors = [
        '.attachment-card', '.file-attachment', '.uploaded-file',
        '.attachment-item', '[data-file-name]', '.file-list-item'
    ];
    
    attachmentSelectors.forEach(function(selector) {
        var elements = document.querySelectorAll(selector);
        if (elements.length > 0) {
            console.log('Found elements for:', selector, elements.length);
        }
    });
    
    // Test 3: Portal detection
    console.log('Portal detected:', typeof parent.angular !== 'undefined');
}

// Run the test
testAttachmentDetection();
```

## Key Points to Remember

1. **`spForm` is NOT native** - Use DOM methods or server-side validation instead
2. **Server-side validation is most reliable** - Use GlideAjax for real attachment checking
3. **Multiple detection methods** - Combine different approaches for robustness
4. **Portal detection** - Always check if running in Service Portal first
5. **Test thoroughly** - Different ServiceNow versions may have different DOM structures

Choose the approach that best fits your ServiceNow version and requirements. The server-side validation method (Solution 2) is recommended for production use as it's the most reliable.