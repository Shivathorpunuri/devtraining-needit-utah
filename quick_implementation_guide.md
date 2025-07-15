# Quick Implementation - Native ServiceNow Portal Attachment Validation

## Step 1: Create Script Include (Server-Side Validation)

**Navigate to:** System Definition > Script Includes

**Name:** `AttachmentValidator`
**Client Callable:** ✅ Check this box
**Script:**

```javascript
var AttachmentValidator = Class.create();
AttachmentValidator.prototype = Object.extendsObject(AbstractAjaxProcessor, {
    
    validateAttachments: function() {
        var reqItemId = this.getParameter('sysparm_req_item_id');
        
        if (!reqItemId) {
            return 'false';
        }
        
        // Check sys_attachment table for any attachments
        var attachment = new GlideRecord('sys_attachment');
        attachment.addQuery('table_name', 'sc_req_item');
        attachment.addQuery('table_sys_id', reqItemId);
        attachment.query();
        
        return attachment.hasNext().toString();
    },
    
    type: 'AttachmentValidator'
});
```

## Step 2: Create Catalog Client Script

**Navigate to:** Service Catalog > Catalog Definitions > Catalog Client Scripts

**Settings:**
- **Name:** Portal Attachment Validation
- **UI Type:** Service Portal  
- **Type:** onSubmit
- **Catalog Item:** [Leave blank for all items, or select specific item]
- **Applies to:** Catalog Item

**Script:**

```javascript
function onSubmit() {
    // Only run in Service Portal
    if (typeof parent.angular !== 'undefined') {
        
        // Use GlideAjax to check attachments on server
        var ga = new GlideAjax('AttachmentValidator');
        ga.addParam('sysparm_name', 'validateAttachments');
        ga.addParam('sysparm_req_item_id', g_form.getUniqueValue());
        ga.getXMLWait();
        
        var result = ga.getAnswer();
        
        if (result === 'false') {
            g_form.addErrorMessage("At least one attachment is required before submitting this request.");
            return false;
        }
    }
    
    return true;
}
```

## Alternative: Simple DOM Detection (Client-Side Only)

If you prefer a simpler approach without server calls:

```javascript
function onSubmit() {
    // Only run in Service Portal
    if (typeof parent.angular !== 'undefined') {
        
        var hasAttachments = false;
        
        // Check for file input elements with files
        var fileInputs = document.querySelectorAll('input[type="file"]');
        for (var i = 0; i < fileInputs.length; i++) {
            if (fileInputs[i].files && fileInputs[i].files.length > 0) {
                hasAttachments = true;
                break;
            }
        }
        
        // Check for attachment display elements
        if (!hasAttachments) {
            var attachmentElements = document.querySelectorAll(
                '.attachment-card, .file-attachment, .uploaded-file, .attachment-item, [data-file-name]'
            );
            hasAttachments = attachmentElements.length > 0;
        }
        
        if (!hasAttachments) {
            g_form.addErrorMessage("Please attach at least one file before submitting.");
            return false;
        }
    }
    
    return true;
}
```

## Testing Your Implementation

1. **Go to your Service Portal**
2. **Navigate to a catalog item**
3. **Try to submit without attachments** - should show error message
4. **Add an attachment and submit** - should work normally

## Troubleshooting

### If the script isn't working:

1. **Check portal detection:**
   Open browser console and type:
   ```javascript
   console.log('Portal detected:', typeof parent.angular !== 'undefined');
   ```

2. **Test attachment detection:**
   ```javascript
   // Check for file inputs
   console.log('File inputs:', document.querySelectorAll('input[type="file"]').length);
   
   // Check for attachment elements
   console.log('Attachment elements:', document.querySelectorAll('.attachment-card, .file-attachment').length);
   ```

3. **Verify Script Include:**
   - Make sure "Client Callable" is checked
   - Test in background script:
   ```javascript
   var validator = new AttachmentValidator();
   gs.print(validator.validateAttachments()); // Should not error
   ```

### For Multiple Attachment Requirements:

Modify the validation function:

```javascript
// In AttachmentValidator Script Include
validateAttachmentsCount: function() {
    var reqItemId = this.getParameter('sysparm_req_item_id');
    var minRequired = parseInt(this.getParameter('sysparm_min_count')) || 1;
    
    var attachment = new GlideRecord('sys_attachment');
    attachment.addQuery('table_name', 'sc_req_item');
    attachment.addQuery('table_sys_id', reqItemId);
    attachment.query();
    
    var count = attachment.getRowCount();
    return (count >= minRequired).toString();
}
```

```javascript
// In Client Script
function onSubmit() {
    if (typeof parent.angular !== 'undefined') {
        var ga = new GlideAjax('AttachmentValidator');
        ga.addParam('sysparm_name', 'validateAttachmentsCount');
        ga.addParam('sysparm_req_item_id', g_form.getUniqueValue());
        ga.addParam('sysparm_min_count', '2'); // Require 2 attachments
        ga.getXMLWait();
        
        if (ga.getAnswer() === 'false') {
            g_form.addErrorMessage("At least 2 attachments are required.");
            return false;
        }
    }
    return true;
}
```

## Why This Works

1. **Server-side validation** - Checks actual attachment records in database
2. **Portal detection** - Only runs in Service Portal, not native UI
3. **GlideAjax** - Uses native ServiceNow APIs for server communication
4. **No third-party dependencies** - Uses only built-in ServiceNow functionality

This solution is **production-ready** and **upgrade-safe** because it only uses standard ServiceNow APIs.