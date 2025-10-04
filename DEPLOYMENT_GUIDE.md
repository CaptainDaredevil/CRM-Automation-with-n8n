# 🚀 CRM Lead Automation - Deployment Guide

## Pre-Deployment Checklist

### 1. Infrastructure Setup

- [ ] **n8n Instance** - Cloud (n8n.cloud) or self-hosted via Docker
- [ ] **Domain/Webhook URL** - For receiving website form submissions
- [ ] **SSL Certificate** - Required for webhook security

### 2. API Credentials & OAuth Setup

Configure OAuth2 credentials for all integrated services:

#### Gmail API
1. Enable Gmail API in [Google Cloud Console](https://console.cloud.google.com)
2. Create OAuth 2.0 credentials (Web application)
3. Add authorized redirect URIs for n8n
4. Configure in n8n: Credentials → Gmail OAuth2

#### Google Sheets API
1. Enable Google Sheets API in Google Cloud Console
2. Use same OAuth credentials as Gmail
3. Share target spreadsheet with service account email

#### Google Forms API
1. Enable Google Forms API
2. Get Form ID from form URL: `forms.google.com/.../FORM_ID/edit`

#### HubSpot API
1. Go to HubSpot → Settings → Integrations → Private Apps
2. Create new app with scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`
3. Copy API key → Add to n8n credentials

#### Slack API
1. Create Slack App at [api.slack.com/apps](https://api.slack.com/apps)
2. Add Bot Token Scopes: `chat:write`, `channels:read`
3. Install app to workspace
4. Copy Bot User OAuth Token → Add to n8n

#### Asana API
1. Go to Asana → Settings → Apps → Developer apps
2. Create Personal Access Token
3. Get Workspace GID from workspace URL or API
4. Get Assignee GID: [Asana API Explorer](https://developers.asana.com/docs/get-users-in-a-workspace)

### 3. Google Sheets Structure

Create a Google Sheet with **TWO tabs**:

**Tab 1: "Leads"** (valid leads)
| Timestamp | Lead ID | Name | Email | Phone | Company | Source | Message | Status |
|-----------|---------|------|-------|-------|---------|--------|---------|--------|

**Tab 2: "Leads_Errors"** (invalid leads for manual review)
| Timestamp | Lead ID | Name | Email | Phone | Company | Source | Message | Validation Error | Status |
|-----------|---------|------|-------|-------|---------|--------|---------|------------------|--------|

**Important:** Headers must match exactly (case-sensitive)

### 4. Environment Variables

1. Copy `.env.example` to `.env`
2. Fill in all required values:

```bash
# Quick setup command
cp .env.example .env
nano .env  # or use your preferred editor
```

### 5. HubSpot Custom Properties

Create these custom contact properties in HubSpot:

| Property Name | Field Type | Description |
|---------------|------------|-------------|
| `lead_source` | Single-line text | Source of lead (Website/Email/Form) |
| `lead_status` | Dropdown | Current lead status |
| `first_contact_date` | Date picker | When lead first contacted |
| `last_contact_date` | Date picker | Most recent contact |
| `initial_message` | Multi-line text | First inquiry message |
| `latest_message` | Multi-line text | Most recent message |

**Setup path:** HubSpot → Settings → Properties → Create property

---

## Fixed Issues (Implemented in v1.1)

### ✅ Webhook Response Timing - FIXED

**Solution Implemented:**
- Added "Is Website Form?" IF node immediately after CRM merge
- Webhook receives HTTP 200 response within ~2-3 seconds
- Email, Slack, Asana, and Sheets operations run asynchronously after response
- Flow: Webhook → Normalize → Validate → CRM → **Quick Response** → [Parallel: Email, Slack, Asana, Sheets]

**Result:** No more timeout issues on slow API calls.

### ✅ Gmail Trigger Broadened - FIXED

**Solution Implemented:**
- Changed from subject filter to label-based monitoring
- Now watches for Gmail label: `Label_Leads`
- Catches all lead emails regardless of subject line

**Setup Required:**
1. Create Gmail label "Leads"
2. Set up Gmail filter to auto-label incoming leads (suggested criteria: from specific domains, contains keywords like "inquiry", "contact", "demo")
3. n8n workflow automatically monitors this label

### ✅ Email Validation - FIXED

**Solution Implemented:**
- Added email regex validation: `/^[^\s@]+@[^\s@]+\.[^\s@]+$/`
- Added name length validation (minimum 2 characters)
- Added "Data Valid?" IF node after normalization
- Invalid leads route to Google Sheets tab: **"Leads_Errors"**
- Valid leads proceed to CRM as normal

**Result:** Clean data in CRM, easy manual review of problematic submissions.

### ✅ Welcome Email Reply Conflict - FIXED

**Solution Implemented:**
- Removed contradictory "do not reply" footer
- Updated to: "This is an automated confirmation message"
- Body encourages replies: "feel free to reply to this email and we'll respond promptly"

**Result:** Consistent messaging that welcomes customer engagement.

### ✅ Schedule Standardized - FIXED

**Decision Made:** Weekly reports run **Tuesday 9 AM** (`0 9 * * 2`)

**Rationale:**
- Captures complete week including weekend leads
- Gives Monday for any system issues to be resolved
- Sales team has fresh data for Tuesday meetings

**All documentation updated** to reflect Tuesday schedule.

### ⚠️ Single-CRM Limitation

**Current:** HubSpot only
**Promised:** HubSpot, Pipedrive, Zoho, Google Sheets/Notion

**Future Enhancement - CRM Abstraction:**

```javascript
// Add after normalization
const crmType = $env.CRM_TYPE || 'hubspot';

// Branch workflow based on CRM_TYPE:
// - hubspot → existing nodes
// - pipedrive → Pipedrive nodes (contact create/update)
// - zoho → Zoho CRM nodes
// - sheets → Google Sheets direct write (no API needed)
```

For MVP, clearly document: "Currently supports HubSpot. Multi-CRM support available on request."

---

## Security & Privacy Considerations

### GDPR Compliance

**Data Collected:**
- Name, email, phone, company name, inquiry message
- Timestamps, source tracking, IP addresses (if webhook captures)

**Retention:**
- Default: Indefinite in Google Sheets + HubSpot
- Recommended: 365-day retention policy

**Data Protection Measures:**

1. **Slack Notifications** - Consider masking sensitive data:
```javascript
// Mask phone numbers in Slack
const maskedPhone = $json.phone ?
  $json.phone.replace(/(\d{3})\d{3}(\d{4})/, '$1***$2') :
  'Not provided';
```

2. **Access Control:**
- Limit Google Sheets sharing (specific users only)
- Enable HubSpot audit logs
- Use Slack private channels for lead notifications

3. **Data Deletion:**
- Document process for GDPR "Right to be Forgotten"
- Create cleanup workflow for leads older than retention period

### Webhook Security

**Recommendations:**
1. Enable webhook authentication in n8n
2. Use API keys or HMAC signatures from form providers
3. Rate limiting to prevent abuse
4. IP whitelisting for known sources

---

## Testing Protocol

### Pre-Production Testing

#### 1. Test Lead Capture Sources

**Webhook Test:**
```bash
curl -X POST https://your-n8n-instance.com/webhook/lead-capture \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test User",
    "email": "test@example.com",
    "phone": "555-0123",
    "company": "Test Corp",
    "message": "This is a test inquiry"
  }'
```

Expected outcome:
- ✅ JSON response: `{"success": true, "lead_id": "LEAD-...", "message": "Lead captured successfully"}`
- ✅ Lead appears in HubSpot
- ✅ Welcome email sent to test@example.com
- ✅ Slack notification posted
- ✅ Asana task created
- ✅ Row added to Google Sheets

**Gmail Test:**
1. Send email to monitored inbox and apply "Leads" label (or let Gmail filter auto-label it)
2. Wait up to 1 minute (polling interval)
3. Verify same outcomes as webhook test

**Google Form Test:**
1. Submit test form response
2. Wait up to 1 minute
3. Verify outcomes

#### 2. Test Duplicate Handling

1. Submit same lead (same email) twice
2. Verify:
   - First submission creates new contact
   - Second submission updates existing contact (no duplicate)
   - Both generate separate notifications/tasks

#### 3. Test Weekly Report

**Manual Trigger:**
1. In n8n, click "Execute Workflow" on "Schedule: Weekly Report" node
2. Verify:
   - Google Doc created with lead summary
   - PDF generated and downloaded
   - Email sent to manager with PDF attachment
   - Report includes statistics by source

#### 4. Test Error Handling

**Invalid Data Test:**
1. Submit lead with invalid email: `notanemail`
2. Verify:
   - Lead appears in "Leads_Errors" tab (not "Leads" tab)
   - Validation Error column shows: "Invalid email format"
   - Status shows: "Requires Review"
   - CRM does NOT receive this lead
   - No welcome email sent (correct behavior)

**Email Bounce Test:**
1. Submit lead with valid format but fake domain: `test@nonexistentdomain12345.com`
2. Verify workflow continues (doesn't fail completely)
3. Check n8n error logs - email send may fail but workflow completes

**API Failure Test:**
1. Temporarily disable Slack credentials
2. Submit test lead
3. Verify: Slack fails but CRM, email, Asana, Sheets still process

---

## Performance Optimization

### Expected Load

| Metric | Estimate |
|--------|----------|
| Leads/day | 10-50 |
| Peak hours | 9 AM - 5 PM business hours |
| Weekly report size | 50-350 leads |
| Execution time/lead | 5-10 seconds |

### Scaling Considerations

**If exceeding 100 leads/day:**
1. Switch from Gmail trigger (1-min polling) to webhook if possible
2. Enable n8n workflow queuing
3. Consider dedicated database (PostgreSQL/Supabase) instead of Google Sheets
4. Implement batch processing for notifications

**If weekly report > 500 leads:**
1. Use Google Sheets query instead of getAll
2. Generate summary stats only, link to full data
3. Consider moving to dedicated reporting tool (Metabase, Google Data Studio)

---

## Monitoring & Maintenance

### Key Metrics to Track

1. **Lead Capture Rate:** Leads captured vs. expected volume
2. **Error Rate:** Failed executions / total executions
3. **Response Time:** Webhook response time
4. **Email Deliverability:** Bounce rate for welcome emails
5. **Task Completion:** % of Asana tasks completed within 24h

### Weekly Maintenance Tasks

- [ ] Review n8n execution logs for errors
- [ ] Check email bounce reports
- [ ] Verify Asana tasks are being assigned and completed
- [ ] Audit Google Sheets for data quality issues
- [ ] Review HubSpot for duplicate contacts

### Monthly Maintenance Tasks

- [ ] Review API usage/quotas (Google, HubSpot, Slack)
- [ ] Update OAuth tokens if expiring
- [ ] Analyze lead source performance
- [ ] Clean up old test data
- [ ] Backup workflow JSON

---

## Rollback Plan

**If critical failure occurs:**

1. **Immediate:** Disable n8n workflow execution
2. **Fallback:** Email leads directly to sales@ address (manual processing)
3. **Investigate:** Review n8n error logs, API status pages
4. **Restore:** Import last known good workflow JSON
5. **Re-enable:** After testing in staging environment

**Backup Strategy:**
```bash
# Export workflow JSON weekly
curl -H "X-N8N-API-KEY: your_api_key" \
  https://your-n8n.com/api/v1/workflows/WORKFLOW_ID \
  > "crm-workflow-backup-$(date +%Y%m%d).json"
```

---

## Support & Troubleshooting

### Common Issues

**"Webhook not receiving data"**
- Check firewall/network settings
- Verify webhook URL is publicly accessible
- Test with curl command
- Check form provider webhook configuration

**"Gmail trigger not firing"**
- Verify OAuth token hasn't expired
- Check that "Leads" label exists in Gmail
- Verify Gmail filter is auto-labeling incoming emails
- Manually apply "Leads" label to test email
- Increase polling frequency temporarily for testing

**"HubSpot contact not created"**
- Check API key permissions
- Verify custom properties exist in HubSpot
- Check execution logs for specific error

**"Weekly report not sent"**
- Verify cron expression is correct for timezone
- Check n8n server timezone settings
- Manually trigger to test

### Debug Mode

Enable n8n debug logging:
```bash
# Docker
docker run -e N8N_LOG_LEVEL=debug n8n

# Cloud
Settings → Log Level → Debug
```

---

## Contact & Documentation

**Technical Documentation:**
- n8n docs: https://docs.n8n.io
- HubSpot API: https://developers.hubspot.com
- Google APIs: https://developers.google.com

**Project Status:** ✅ Production-Ready v1.1

**Changelog:**
- v1.1 (2025-10-04): Fixed webhook timing, added validation, broadened Gmail trigger, fixed email copy
- v1.0 (2025-10-04): Initial release

**Last Updated:** 2025-10-04
