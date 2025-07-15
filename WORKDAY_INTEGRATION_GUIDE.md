# Workday to ServiceNow Employee Integration

This integration provides REST API endpoints to receive hired employee details from Workday and create or update employee records in ServiceNow.

## API Endpoints

### Base URL
```
https://your-instance.service-now.com/api/x_58872_needit/workday_employee_integration
```

### Authentication
- Use ServiceNow basic authentication or OAuth
- Ensure the calling service has appropriate permissions to access the REST API

## Endpoints

### 1. Create/Update Single Employee
**POST** `/employee`

Creates a new employee record or updates an existing one based on employee_id or email.

#### Request Format
```json
{
  "employee_id": "WD001234",
  "username": "john.doe",
  "email": "john.doe@company.com",
  "first_name": "John",
  "last_name": "Doe",
  "title": "Software Engineer",
  "department": "Information Technology",
  "location": "New York Office",
  "phone": "+1-555-123-4567",
  "mobile_phone": "+1-555-987-6543",
  "start_date": "2024-01-15",
  "manager_employee_id": "WD005678",
  "manager_email": "jane.manager@company.com"
}
```

#### Response Format
```json
{
  "success": true,
  "message": "Employee created successfully",
  "user_sys_id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
  "employee_id": "WD001234",
  "action": "created"
}
```

### 2. Batch Create/Update Employees
**POST** `/employees/batch`

Processes multiple employee records in a single request for efficient bulk operations.

#### Request Format
```json
{
  "employees": [
    {
      "employee_id": "WD001234",
      "username": "john.doe",
      "email": "john.doe@company.com",
      "first_name": "John",
      "last_name": "Doe",
      "title": "Software Engineer",
      "department": "Information Technology",
      "location": "New York Office",
      "phone": "+1-555-123-4567",
      "start_date": "2024-01-15",
      "manager_employee_id": "WD005678"
    },
    {
      "employee_id": "WD001235",
      "username": "jane.smith",
      "email": "jane.smith@company.com",
      "first_name": "Jane",
      "last_name": "Smith",
      "title": "Product Manager",
      "department": "Product",
      "location": "San Francisco Office",
      "phone": "+1-555-123-4568",
      "start_date": "2024-01-16",
      "manager_employee_id": "WD005679"
    }
  ]
}
```

#### Response Format
```json
{
  "success": true,
  "message": "Batch processing completed",
  "summary": {
    "total_processed": 2,
    "successful": 2,
    "errors": 0
  },
  "results": [
    {
      "index": 0,
      "employee_id": "WD001234",
      "email": "john.doe@company.com",
      "success": true,
      "user_sys_id": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6",
      "action": "created"
    },
    {
      "index": 1,
      "employee_id": "WD001235",
      "email": "jane.smith@company.com",
      "success": true,
      "user_sys_id": "b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7",
      "action": "created"
    }
  ]
}
```

## Field Mapping

| Workday Field | ServiceNow Field | Required | Notes |
|---------------|------------------|----------|-------|
| employee_id | employee_number | Yes | Primary identifier |
| email | email | Yes | Used for duplicate detection |
| username | user_name | No | Auto-generated if not provided |
| first_name | first_name | No | |
| last_name | last_name | No | |
| title / job_title | title | No | |
| department | department | No | Creates department if not exists |
| location | location | No | Creates location if not exists |
| phone / work_phone | phone | No | |
| mobile_phone | mobile_phone | No | |
| start_date / hire_date | u_start_date | No | |
| manager_employee_id | manager | No | Links to manager record |
| manager_email | manager | No | Alternative manager lookup |

## Implementation Features

### 1. Duplicate Detection
- First checks by `employee_id` (employee_number field)
- If not found, checks by `email` address
- Updates existing record if found, creates new if not found

### 2. Manager Assignment
- Supports lookup by manager's employee_id or email
- Manager must exist in ServiceNow before assignment
- Gracefully handles missing manager references

### 3. Department and Location Handling
- Automatically creates departments and locations if they don't exist
- Logs warnings for missing reference data
- Uses name-based matching for existing records

### 4. Role Assignment
- New employees automatically get `x_58872_needit.needit_user` role
- Existing users keep their current roles
- Additional roles can be assigned manually

### 5. Error Handling
- Comprehensive validation of required fields
- Detailed error messages for debugging
- Batch processing continues on individual failures
- All activities are logged for audit trail

## HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 400 | Bad Request - Missing required fields or invalid format |
| 500 | Internal Server Error - Processing error |

## Security Considerations

1. **Authentication**: Ensure proper authentication is configured
2. **ACLs**: Verify access control lists allow API access
3. **Field Security**: Review field-level security for sensitive data
4. **Audit**: All activities are logged in ServiceNow logs

## Testing

### Single Employee Test
```bash
curl -X POST \
  'https://your-instance.service-now.com/api/x_58872_needit/workday_employee_integration/employee' \
  -H 'Authorization: Basic <base64-encoded-credentials>' \
  -H 'Content-Type: application/json' \
  -d '{
    "employee_id": "TEST001",
    "email": "test.user@company.com",
    "first_name": "Test",
    "last_name": "User",
    "title": "Test Engineer",
    "department": "Testing"
  }'
```

### Batch Test
```bash
curl -X POST \
  'https://your-instance.service-now.com/api/x_58872_needit/workday_employee_integration/employees/batch' \
  -H 'Authorization: Basic <base64-encoded-credentials>' \
  -H 'Content-Type: application/json' \
  -d '{
    "employees": [
      {
        "employee_id": "TEST001",
        "email": "test1@company.com",
        "first_name": "Test",
        "last_name": "User1"
      },
      {
        "employee_id": "TEST002", 
        "email": "test2@company.com",
        "first_name": "Test",
        "last_name": "User2"
      }
    ]
  }'
```

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   - Verify credentials and permissions
   - Check if REST API access is enabled

2. **Field Validation Errors**
   - Ensure employee_id and email are provided
   - Verify email format is valid

3. **Department/Location Not Found**
   - Check if auto-creation is working
   - Verify naming conventions match

4. **Manager Assignment Fails**
   - Ensure manager exists in ServiceNow first
   - Check employee_id or email accuracy

### Logs
Check ServiceNow logs for detailed error information:
- Navigate to System Logs > System Log > All
- Filter by source: "WorkdayIntegration"

## Maintenance

1. **Monitor Performance**: Track API response times and batch sizes
2. **Review Logs**: Regular log review for errors and warnings
3. **Update Mappings**: Adjust field mappings as business requirements change
4. **Security Review**: Periodic review of access permissions

## Support

For technical support or questions about this integration:
1. Check the ServiceNow logs for detailed error messages
2. Review the field mappings in this document
3. Test with single employee records before batch processing
4. Contact your ServiceNow administrator for access or permission issues