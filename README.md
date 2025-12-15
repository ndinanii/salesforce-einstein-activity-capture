# salesforce einstein activity capture

# Einstein Activity Capture: Implementation Issues & Solutions

## Document Overview

This document details critical issues encountered during the implementation of Einstein Activity Capture (EAC) in a Salesforce Developer Organization. Each problem is documented with technical root causes, symptoms, and resolutions suitable for both developers and Salesforce administrators.

**Environment**: Salesforce Developer Edition  
**User**: Wonga Myendeki  
**Integration**: Gmail via Einstein Activity Capture  
**Date**: December 2024

---

## Problem 1: Licensing Configuration Error

### Severity
**Critical** - Complete system failure; no sync possible

### Issue Summary
The Einstein Activity Capture feature was enabled at the organization level, but the system completely ignored all user-level configurations. Despite following standard enablement procedures, the feature remained non-functional with no sync activity initiated.

### Symptoms
- **Health Metrics Dashboard**: Licensed Users count displayed as `0`
- **User Settings**: Configuration options were visible but had no effect
- **System Behavior**: No sync processes initiated despite proper Gmail connectivity
- **User Status**: User appeared in EAC configuration but showed as "unlicensed"

### Root Cause Analysis

**Technical Explanation**:  
Einstein Activity Capture requires explicit permission set assignment at the user level, separate from the org-level feature enablement. The system architecture uses a two-tier permission model:

1. **Org-Level Enablement**: Activates the feature infrastructure (APIs, sync services, storage)
2. **User-Level Permission Set**: Grants individual users access to sync capabilities

The critical permission set required is: **`Standard Einstein Activity Capture User`**

Without this permission set assignment:
- The sync engine ignores the user during batch processing
- OAuth connections cannot be established
- User quotas are not allocated
- The user does not appear in licensing counts

**Why This Happens**:  
Salesforce separates feature availability from user entitlement to allow granular control over which users consume EAC licenses. In Developer Orgs, this permission must be manually assigned as there is no automatic provisioning.

### Resolution Steps

**For Administrators**:
1. Navigate to **Setup** (gear icon) → Quick Find → `Permission Sets`
2. Search for and select: **`Standard Einstein Activity Capture User`**
3. Click **Manage Assignments**
4. Click **Add Assignments**
5. Select the user(s) requiring EAC access (e.g., `Wonga Myendeki`)
6. Click **Assign** → **Done**
7. Verify assignment in **Health Metrics** dashboard (refresh after 2-3 minutes)

**For Developers**:
```apex
// Verify permission set assignment programmatically
PermissionSetAssignment[] assignments = [
    SELECT Id, PermissionSet.Name, Assignee.Name 
    FROM PermissionSetAssignment 
    WHERE PermissionSet.Name = 'Standard_Einstein_Activity_Capture_User'
    AND AssigneeId = :UserInfo.getUserId()
];

System.debug('EAC Permission Assigned: ' + !assignments.isEmpty());
```

**Alternative: Profile-Level Assignment** (Not Recommended):
- Navigate to **Setup** → **Profiles** → [User's Profile]
- Edit and enable "Einstein Activity Capture" under Administrative Permissions
- Note: This grants access to all users with that profile; permission sets offer better granularity

### Validation
After resolution, verify:
- ✅ Health Metrics shows `Licensed Users: 1` (or appropriate count)
- ✅ User appears in "Manage Users" under EAC settings
- ✅ User Health Status shows "Licensed: Yes"

### Prevention
**Configuration Checklist**:
- [ ] Org-level EAC enabled
- [ ] Permission Set assigned to user
- [ ] User has valid Gmail/Outlook account
- [ ] User profile allows "Send Email" permission
- [ ] Lightning Experience enabled (EAC requires Lightning)

---

## Problem 2: Authentication Failure (Unlinked Account)

### Severity
**High** - Feature is licensed but non-functional; no data sync occurs

### Issue Summary
Despite successfully enabling Einstein Activity Capture and assigning the correct permission set, the system could not sync any email or calendar data. The connection between Salesforce and the user's Gmail account was not established.

### Symptoms
- **User Health Status Page**: Displayed `Google account connected to Salesforce: Unlinked`
- **Sync Status**: Showed `Not started` indefinitely
- **OAuth Status**: No active token present
- **Error Messages**: None visible in UI; silent failure
- **Sync Dashboard**: No activities captured; 0 emails/events synced

### Root Cause Analysis

**Technical Explanation**:  
Einstein Activity Capture uses a **User-Level OAuth 2.0** authentication model. This means:

1. **Org-Level Setup is Insufficient**: Enabling EAC at the organization level only configures the Salesforce infrastructure; it does NOT create individual OAuth connections
2. **User Consent Required**: Each user must explicitly authorize Salesforce to access their Gmail/Google Calendar data via an OAuth handshake
3. **No Automatic Linking**: Unlike some integrations, EAC does not auto-detect or auto-link email accounts based on user email fields in Salesforce

**OAuth Flow Architecture**:
```
User Action → Salesforce OAuth Request → Google Authorization Server
                                       ↓
                                  User Consent Screen
                                       ↓
                           Authorization Code Returned
                                       ↓
                    Salesforce Exchanges Code for Access Token
                                       ↓
                              Token Stored Securely
                                       ↓
                         Sync Engine Activates for User
```

**Why This Design Exists**:
- **Security**: Users must explicitly consent to third-party access
- **Compliance**: Meets GDPR and privacy requirements (user controls their data)
- **Flexibility**: Allows users to connect different email accounts than their Salesforce login

### Resolution Steps

**For End Users**:
1. Click your **profile picture** (top right) → **Settings**
2. In Quick Find, search: `Einstein Activity Capture`
3. Click **Einstein Activity Capture Settings**
4. Locate the prompt: **"Connect Account"** or **"Google account: Unlinked"**
5. Click **Connect Account**
6. **Google OAuth Screen** will appear:
   - Sign in to your Gmail account (if not already signed in)
   - Review permissions requested:
     - Read, compose, send, and delete email
     - Manage calendar events
   - Click **Allow**
7. You'll be redirected back to Salesforce
8. Verify status changes to: **"Google account connected to Salesforce: Linked"**

**For Administrators Assisting Users**:
1. Navigate to **Setup** → **Einstein Activity Capture** → **Manage Users**
2. Locate the user (e.g., `Wonga Myendeki`)
3. Check **Connection Status** column
4. If showing "Unlinked", guide the user through the OAuth flow above
5. **Note**: Admins cannot perform OAuth on behalf of users due to security policies

**For Developers** (Testing/Debugging):
```javascript
// Check OAuth token status via REST API
GET /services/data/v59.0/einstein/activity-capture/settings/users/{userId}

// Response includes:
{
  "isConnected": false,
  "connectionStatus": "UNLINKED",
  "lastSyncTime": null,
  "oauthStatus": "NOT_AUTHORIZED"
}
```

**Troubleshooting OAuth Issues**:

| Problem | Cause | Solution |
|---------|-------|----------|
| OAuth popup blocked | Browser popup blocker | Allow popups for Salesforce domain |
| "Invalid Scope" error | Gmail API not enabled | Enable Gmail API in Google Cloud Console |
| "Access Denied" | User clicked "Deny" | Re-attempt connection; ensure user understands permissions |
| Connection lost after weeks | Token expired | Reconnect account; enable long-lived refresh tokens |

### Validation
After resolution, verify:
- ✅ User Health Status: `Google account connected: Linked`
- ✅ Sync Status: Changes from `Not started` to `In Progress` or `Active`
- ✅ OAuth token present in background (not visible to user, but sync begins)
- ✅ Historical Harvest process starts (may take several hours)

### Prevention
**User Onboarding Checklist**:
1. [ ] User notified that OAuth connection is required
2. [ ] User has valid Gmail credentials
3. [ ] User understands permission scope (email + calendar access)
4. [ ] Browser allows popups from Salesforce
5. [ ] User completes OAuth within setup session (token doesn't expire during setup)

**Admin Monitoring**:
- Regularly check **Manage Users** page for unlinked accounts
- Set up alerts/reports for users with `Connection Status = Unlinked` for >24 hours
- Document OAuth completion as part of user onboarding process

---

## Problem 3: Reporting "False Positive" (No Recent Sync Times)

### Severity
**Low** - Cosmetic issue; does not impact functionality but causes user confusion

### Issue Summary
The **Sync Time Report** displayed alarming red text stating: `"No recent sync times were found."` This appeared to indicate a system failure, causing concern that the Einstein Activity Capture configuration was broken or not functioning correctly.

### Symptoms
- **Sync Time Report**: Red error text: "No recent sync times were found."
- **User Perception**: Belief that sync is failing or not configured properly
- **Other Metrics**: All other health indicators showed normal/healthy status
- **Timeline**: Issue appeared immediately after fresh installation (<1 hour old)

### Root Cause Analysis

**Technical Explanation**:  
The Sync Time Report is designed to calculate and display **average sync performance metrics over a 7-day rolling window**. The report logic:

1. Queries sync job completion timestamps from the past 7 days
2. Calculates average time per sync cycle
3. Displays trends (improving/degrading performance)
4. Highlights anomalies (unusually slow syncs)

**Why the "Error" Appeared**:
- **Fresh Installation**: The org had EAC enabled for less than 1 hour
- **No Historical Data**: Zero sync cycles had completed in the past 7 days
- **Report Logic**: The report cannot calculate an average with no data points
- **Display Behavior**: Red text is shown when query returns 0 results (by design)

**This is NOT an error**. The report is functioning correctly; it simply cannot display metrics that don't yet exist.

**Mathematical Context**:
```
Average Sync Time = (Sum of all sync durations) / (Number of completed syncs)

When: Number of completed syncs = 0
Then: Average Sync Time = undefined (cannot divide by zero)
Display: "No recent sync times were found."
```

### Resolution Steps

**For Users/Administrators**:
1. **Recognize as Standard Behavior**: This is expected for new installations
2. **Wait for Data Accumulation**: Sync time data will populate after:
   - First sync cycle completes (typically 4-8 hours after OAuth connection)
   - Historical Harvest finishes processing past emails
   - At least 2-3 sync cycles complete (for meaningful averages)
3. **Check Again in 24-48 Hours**: Report will populate once sync history exists
4. **Focus on Other Metrics**: Use "User Health Status" and "Sync Status" for immediate validation

**For Developers** (Understanding the Report Logic):
```sql
-- Simplified query behind the Sync Time Report
SELECT AVG(SyncDurationMinutes) 
FROM EinsteinActivityCaptureSyncJob__c
WHERE CompletedDate >= LAST_N_DAYS:7
  AND Status = 'Completed'
  AND UserId = :currentUserId

-- When fresh install:
-- Result: 0 rows returned
-- UI displays: "No recent sync times were found."
```

**What to Monitor Instead** (First 48 Hours):
1. **User Health Status**: Should show "Licensed: Yes" and "Connected: Linked"
2. **Sync Status**: Should progress from "Not started" → "In Progress" → "Active"
3. **Historical Harvest**: Monitor contact/activity counts increasing
4. **Manual Test**: Send a test email and verify it appears in timeline (after delay)

### Validation
After 24-48 hours, verify:
- ✅ Sync Time Report shows actual metrics (green/yellow indicators)
- ✅ Average sync time displayed (e.g., "4.2 minutes per cycle")
- ✅ Trend graph appears showing historical performance
- ✅ No red text (unless genuine performance issue exists)

### Prevention
**User Communication Template**:
> "The 'No recent sync times' message is normal for new setups. This report requires 7 days of sync history to calculate averages. Check back in 24-48 hours to see your sync performance metrics. In the meantime, use the 'User Health Status' page to verify your connection is active."

**Documentation Best Practice**:
- Include this behavior in user guides with clear explanation
- Add a tooltip/help text in custom dashboards explaining the 7-day window
- Set expectations during onboarding that reports populate gradually

---

## Problem 4: Initial Data Latency (Expectation vs. Reality)

### Severity
**Medium** - Causes user confusion and doubt; impacts adoption if not explained properly

### Issue Summary
After successfully configuring Einstein Activity Capture and sending a test email from Gmail to an existing Salesforce contact, the activity did not appear on the contact's timeline even after 10 minutes. This created the perception that the sync was broken or not functioning, leading to troubleshooting efforts that were ultimately unnecessary.

### Symptoms
- **Test Email Sent**: User sent email from Gmail to a contact in Salesforce
- **Timeline Check**: Refreshed contact record repeatedly over 10 minutes
- **No Activity Visible**: Email did not appear in Activity Timeline or related list
- **User Assumption**: "Sync is not working" or "Configuration is broken"
- **Actual Status**: System was functioning correctly but processing historical data first

### Root Cause Analysis

**Technical Explanation**:  
Einstein Activity Capture does not sync new emails in real-time. The sync architecture uses a multi-stage batch processing system with specific prioritization logic:

**Sync Process Stages**:
1. **Historical Harvest** (First Priority)
   - Scans the past 30 days of email history
   - Matches emails to existing Salesforce contacts/leads
   - Can take 4-24 hours depending on email volume
   - Processes oldest emails first, working forward chronologically

2. **Ongoing Sync** (Second Priority)
   - Monitors new emails/calendar events
   - Runs on a scheduled batch cycle (every 4-8 hours by default)
   - Only begins after Historical Harvest completes initial pass

3. **Real-Time Updates** (Not Supported)
   - EAC is NOT a real-time integration
   - No webhooks or instant push notifications
   - Designed for overnight/background processing

**Why the 10-Minute Wait Failed**:
- **Compounding Factors**: Both Problems #1 and #2 (licensing + OAuth) were still unresolved during the initial test
- **No Connectivity**: Without proper licensing and OAuth, no sync process could begin
- **Historical Backlog**: Once connectivity was fixed, the system prioritized scanning 30 days of past emails before processing the brand-new test email
- **Batch Timing**: Even with connectivity, the next sync cycle may not run for 4-8 hours

**Architecture Diagram**:
```
Gmail Inbox → OAuth Connection → Salesforce Sync Engine
                                          ↓
                                  ┌─────────────────┐
                                  │ Job Queue       │
                                  │ 1. Historical   │ ← Runs First
                                  │ 2. Ongoing      │ ← Runs After
                                  │ 3. Calendar     │
                                  └─────────────────┘
                                          ↓
                                  Batch Processing
                                  (Every 4-8 hours)
                                          ↓
                                  Activity Timeline
```

### Resolution Steps

**Immediate Actions Taken**:
1. **Fixed Prerequisites**: Resolved Problems #1 (licensing) and #2 (OAuth)
2. **Waited for Initial Sync**: Allowed Historical Harvest to complete first pass
3. **Result**: System immediately synced 4 contacts once pipeline was open
4. **Validation**: Confirmed sync was working; latency was expected behavior

**For Users Experiencing This**:
1. **Set Correct Expectations**: 
   - First sync can take 4-24 hours
   - New emails may take 4-8 hours to appear
   - This is NOT real-time; designed for batch processing
2. **Check Sync Status**: Navigate to **Einstein Activity Capture Settings** → verify "Sync Status: In Progress"
3. **Monitor Historical Progress**: Check "Contacts Synced" count increasing over time
4. **Test After 24 Hours**: Send test email, wait 24 hours, then check timeline
5. **Use Sync Time Report** (after 7 days): Understand typical sync cadence for your org

**For Administrators**:
1. **Adjust Sync Frequency** (if needed):
   - Setup → Einstein Activity Capture → Configuration
   - Review "Sync Interval" settings (may not be adjustable in all orgs)
2. **Monitor Sync Jobs**:
   - Setup → Apex Jobs → Search for "Einstein Activity Capture"
   - Review job completion times and status
3. **Educate Users**:
   - Document expected latency in training materials
   - Explain Historical Harvest process during onboarding

**For Developers** (Understanding Batch Jobs):
```apex
// Query recent EAC sync jobs
List<AsyncApexJob> jobs = [
    SELECT Id, Status, JobType, CompletedDate, TotalJobItems, JobItemsProcessed
    FROM AsyncApexJob
    WHERE JobType = 'BatchApex'
    AND ApexClass.Name LIKE '%Einstein%Activity%'
    ORDER BY CreatedDate DESC
    LIMIT 10
];

for (AsyncApexJob job : jobs) {
    System.debug('Job Status: ' + job.Status + 
                 ' | Processed: ' + job.JobItemsProcessed + '/' + job.TotalJobItems);
}
```

### Validation
After 24 hours, verify:
- ✅ Historical Harvest completed (check "Last Sync Time" in User Health)
- ✅ Test email now visible on contact timeline
- ✅ Email details accurate (subject, body preview, timestamp)
- ✅ Ongoing sync cycles running regularly (check Apex Jobs)

### Prevention
**User Training Content**:
Create a clear guide explaining:
- **What EAC Is**: A background sync tool, not a real-time integration
- **Expected Timing**: 
  - Historical data: 4-24 hours
  - New emails: 4-8 hours after sending
  - Calendar events: Next sync cycle
- **When to Worry**: If no activity appears after 48 hours, contact admin

**Configuration Best Practices**:
1. **Enable Email Logging** (separate feature) for immediate tracking:
   - Setup → Activity Settings → Enable "Log Emails to Salesforce"
   - This provides instant logging while EAC handles background sync
2. **Set User Expectations**: During setup, explicitly state sync is NOT real-time
3. **Document Sync Schedule**: Let users know when batch jobs typically run (e.g., "overnight" or "every 6 hours")

**Sample User Communication**:
> "Einstein Activity Capture syncs your emails in the background, similar to how your phone backs up photos overnight. After connecting your Gmail, please allow 24 hours for the initial sync to complete. New emails will appear on contact records within 4-8 hours after you send them. This is normal behavior and ensures the system runs efficiently without slowing down your Salesforce experience."

---

## Summary: Problem Dependency Chain

The four problems were interconnected and needed to be resolved in sequence:

```
Problem 1 (Licensing) → Problem 2 (OAuth) → Problem 4 (Latency) → Problem 3 (Reporting)
       ↓                        ↓                    ↓                      ↓
   BLOCKER              BLOCKER              EXPECTATION           COSMETIC
   Must fix first       Must fix second      Requires patience     Resolves with time
```

**Resolution Order**:
1. ✅ Assign permission set (Problem 1)
2. ✅ Complete OAuth connection (Problem 2)
3. ⏳ Wait for Historical Harvest (Problem 4)
4. ⏳ Wait 7 days for metrics (Problem 3)

---

## Key Takeaways for Implementation Teams

### For Administrators
- Always assign permission sets before expecting functionality
- User-level OAuth is mandatory; org-level enablement is insufficient
- Set realistic expectations: first sync takes 24+ hours, not minutes
- Monitor Health Metrics dashboard during first week of rollout
- Document that red "No recent sync times" is normal for new installs

### For Developers
- Test OAuth flows in sandbox before production deployment
- Build monitoring dashboards that distinguish "not yet synced" from "sync failed"
- Consider programmatic permission set assignment for bulk user enablement
- Log sync job statuses for troubleshooting
- Avoid promises of real-time sync in custom apps built on EAC

### For End Users
- Understand that EAC is a background process, not instant
- Complete OAuth connection immediately after admin enables the feature
- Don't panic if reports show errors on day 1; give it 48 hours
- Use Activity Timeline as the source of truth, not external email clients
- Report persistent issues only after 48-hour waiting period

---

## Appendix: Health Check Command List

**Quick Diagnostics** (Run These in Order):

1. **Check Licensing**:
   - Setup → Einstein Activity Capture → Health Metrics
   - Expected: Licensed Users > 0

2. **Check OAuth**:
   - Personal Settings → Einstein Activity Capture Settings
   - Expected: "Google account: Linked"

3. **Check Sync Status**:
   - Personal Settings → Einstein Activity Capture Settings
   - Expected: "Sync Status: In Progress" or "Active"

4. **Check Recent Jobs**:
   - Setup → Apex Jobs → Filter by "Einstein"
   - Expected: Jobs with "Completed" status

5. **Check Activity Data**:
   - Open any Contact record → Activity Timeline
   - Expected: Email activities visible (after 24+ hours)

**If All Checks Pass**: System is healthy; latency is expected.  
**If Any Check Fails**: Refer to the relevant problem section above.

---

**Document Version**: 1.0  
**Last Updated**: December 2024  
**Contributors**: Wonga Myendeki (Implementation), Technical Documentation Team
