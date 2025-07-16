# Incident Hold Automation

This ServiceNow solution automatically closes incidents that have been on hold for 3 or more days and sends notifications to the assigned user.

## Components

### 1. Scheduled Script: Auto Close On Hold Incidents After 3 Days
**File:** `update/sysauto_script_hold_incident_check.xml`

- **Purpose:** Runs daily to identify incidents that have been on hold for 3+ days
- **Schedule:** Daily at 2:00 AM
- **Logic:** 
  - Queries incidents with state = 6 (On Hold)
  - Filters for incidents updated 3+ days ago
  - Only processes active incidents
  - Triggers event `x_58872_needit.closeHoldIncident` for each qualifying incident

### 2. Event Script Action: Close Hold Incident and Notify Assignee
**File:** `update/sysevent_script_action_close_hold_incident.xml`

- **Purpose:** Responds to the `closeHoldIncident` event
- **Actions:**
  - Sets incident state to 7 (Closed)
  - Adds resolution notes explaining automatic closure
  - Sets close code to "Closed by System"
  - Sends email notification to assignee if email exists
  - Logs all activities for audit trail

### 3. Business Rule: Log Incident On Hold
**File:** `update/sys_script_business_rule_hold_incident_log.xml`

- **Purpose:** Tracks when incidents are put on hold
- **Trigger:** When incident state changes to 6 (On Hold)
- **Actions:**
  - Adds work note documenting the hold reason
  - Warns about automatic closure after 3 days
  - Logs the event for monitoring

## Email Notification Details

When an incident is automatically closed, the assignee receives an email with:
- Incident number and short description
- Explanation of automatic closure
- Instructions for reopening if needed
- Timestamp of closure

## Installation

1. Import all XML files into your ServiceNow instance
2. Verify the scheduled script is active and properly configured
3. Test the functionality with a test incident

## Configuration Options

### Modify Hold Duration
To change the 3-day hold period, update the line in the scheduled script:
```javascript
threeDaysAgo.addDaysUTC(-3); // Change -3 to desired number of days
```

### Customize Email Template
Modify the email body in the event script action to match your organization's requirements.

### Adjust Schedule
Change the run time in the scheduled script properties to suit your maintenance windows.

## Monitoring and Reporting

- Check system logs for processing information
- Monitor work notes on incidents for hold tracking
- Create reports on automatically closed incidents using close code "Closed by System"

## Best Practices

1. **Communication:** Inform users about the 3-day auto-closure policy
2. **Documentation:** Ensure hold reasons are properly documented
3. **Monitoring:** Regularly review automatically closed incidents
4. **Escalation:** Consider escalation paths for critical incidents before auto-closure

## Troubleshooting

### Scheduled Script Not Running
- Verify the script is active
- Check system scheduler health
- Review system logs for errors

### Notifications Not Sending
- Verify assignee has valid email address
- Check email configuration settings
- Review SMTP settings

### Incidents Not Closing
- Verify event registration is active
- Check for script errors in logs
- Ensure proper permissions for system user

## Security Considerations

- The scheduled script runs as System Administrator
- Email notifications include incident details
- Ensure appropriate access controls on related tables