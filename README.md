# SOCRadar Incident Connector for Microsoft Sentinel

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Forcunsami%2Fsocradar-sentinel-alarm-connector-api-fork-orcun%2Fmain%2Ftemplate.json)

Automatically create Microsoft Sentinel incidents from SOCRadar alarms in real-time using webhook integration.

## Architecture

This Logic App receives SOCRadar alarm notifications via HTTP webhook and creates corresponding incidents in Microsoft Sentinel:

1. **HTTP Webhook Trigger** - Receives real-time JSON payload from SOCRadar
2. **For Each Loop** - Processes each alarm in the payload
3. **Create Incident** - Creates Sentinel incident with mapped fields

**Benefits:**
- Real-time incident creation (instant)
- Automatic field mapping (title, severity, description)
- Null-safe schema (handles missing fields gracefully)

## Prerequisites

- Azure subscription with Microsoft Sentinel enabled
- Existing Log Analytics Workspace with Sentinel
- Azure account with **Microsoft Sentinel Contributor** role

## Deployment

### Option 1: Portal (Recommended)

Click the **Deploy to Azure** button above and fill in the parameters.

### Option 2: Azure CLI

```bash
az deployment group create \
  --resource-group <YOUR_RESOURCE_GROUP> \
  --template-file template.json \
  --parameters \
    logicAppName="SOCRadar-Sentinel-Webhook-LA" \
    sentinelConnectionName="azuresentinel-webhook-1" \
    sentinelWorkspacePath="/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.OperationalInsights/workspaces/<WORKSPACE_NAME>/providers/Microsoft.SecurityInsights"
```

### Option 3: PowerShell

```powershell
New-AzResourceGroupDeployment `
  -ResourceGroupName <YOUR_RESOURCE_GROUP> `
  -TemplateFile template.json `
  -logicAppName "SOCRadar-Sentinel-Webhook-LA" `
  -sentinelConnectionName "azuresentinel-webhook-1" `
  -sentinelWorkspacePath "/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.OperationalInsights/workspaces/<WORKSPACE_NAME>/providers/Microsoft.SecurityInsights"
```

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| logicAppName | No | SOCRadar-Sentinel-Webhook-LA | Logic App resource name |
| location | No | Resource group location | Azure region |
| sentinelConnectionName | No | azuresentinel-webhook-1 | API connection resource name |
| sentinelWorkspacePath | Yes | - | Full path to Sentinel workspace |

### Sentinel Workspace Path Format

**Required format:**
```
/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.OperationalInsights/workspaces/<WORKSPACE_NAME>/providers/Microsoft.SecurityInsights
```

**How to find:**
1. Azure Portal → Log Analytics workspaces → Your workspace
2. Copy **Subscription ID**, **Resource Group**, **Workspace Name**
3. Build the path using format above

**Example:**
```
/subscriptions/18685d03-5a92-4298-b8d6-7b9264fb878f/resourceGroups/company_resource_group_0001/providers/Microsoft.OperationalInsights/workspaces/company-log-analytics-workspace-0001/providers/Microsoft.SecurityInsights
```

## Post-Deployment Configuration

### Step 1: Authorize API Connection

The managed connection needs permissions to create incidents.

1. Azure Portal → API Connections → `azuresentinel-webhook-1`
2. Click **Edit API connection**
3. Click **Authorize**
4. Sign in with an account that has **Microsoft Sentinel Contributor** role

### Step 2: Configure Webhook in SOCRadar

1. Azure Portal → Logic Apps → `SOCRadar-Sentinel-Webhook-LA`
2. Overview → Copy **Workflow URL** (starts with `https://prod-...logic.azure.com/...`)
3. SOCRadar Platform → Settings → Webhooks
4. Add the Logic App URL as webhook endpoint
5. Configure SOCRadar to send alarm notifications to this URL

## Field Mapping

| SOCRadar Field | Sentinel Incident Field | Notes |
|----------------|-------------------------|-------|
| `alarm_id` | Title (prefix: SOCRadar-AlarmID-) | Used for bidirectional sync |
| `alarm_type_details.alarm_generic_title` | Title | Alarm description |
| `alarm_risk_level` | Severity | Mapped: INFO→Informational, LOW→Low, MEDIUM→Medium, HIGH→High |
| `alarm_text` | Description | Alarm details |
| `alarm_created_date` | Created Time | Timestamp |
| `alarm_details` | Link | SOCRadar platform URL |

**Title Format:** `SOCRadar-AlarmID-{alarm_id}`

Example: `SOCRadar-AlarmID-77861826`

This format enables bidirectional sync with the [polling integration](https://github.com/orcunsami/azure-bidirectional-incident-app-fork-orcun).

## Testing

### Verify Deployment

```bash
# Check Logic App status
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-Sentinel-Webhook-LA \
  --query "state"

# Expected: "Enabled"
```

### Get Webhook URL

```bash
az logic workflow show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-Sentinel-Webhook-LA \
  --query "accessEndpoint" -o tsv
```

### Test Webhook Manually

```bash
WEBHOOK_URL="<YOUR_WEBHOOK_URL>"

curl -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "data": [{
      "alarm_id": 12345,
      "alarm_type_details": {
        "alarm_generic_title": "Test Alarm"
      },
      "alarm_risk_level": "HIGH",
      "alarm_text": "Test alarm description",
      "alarm_created_date": "2025-11-17T10:00:00Z",
      "alarm_details": "https://platform.socradar.com/alarms/12345",
      "status": "OPEN"
    }]
  }'
```

### Verify Incident Created

1. Azure Portal → Microsoft Sentinel → Incidents
2. Look for incident titled: `SOCRadar-AlarmID-12345`
3. Verify severity, description, and timestamps

## Monitoring

### View Run History

**Portal:**
1. Logic App → Overview → Runs history
2. Click on any run for details

**CLI:**
```bash
az logic workflow run list \
  --resource-group <YOUR_RG> \
  --name SOCRadar-Sentinel-Webhook-LA \
  --top 10 \
  --query "[].{name:name, status:status, startTime:startTime}" \
  --output table
```

### Check for Errors

```bash
az logic workflow run show \
  --resource-group <YOUR_RG> \
  --name SOCRadar-Sentinel-Webhook-LA \
  --name <RUN_NAME>
```

## Troubleshooting

### Logic App Run Fails (401/403)

**Cause:** API connection not authorized

**Solution:**
1. Go to API Connection resource
2. Edit API connection
3. Click Authorize and sign in

### Logic App Fails in For Each

**Cause:** SOCRadar payload missing `data` array

**Solution:**
1. Check Logic App run history → trigger output
2. Verify SOCRadar is sending correct JSON format
3. Ensure `data` array exists in payload

### Sentinel Workspace Path Error

**Cause:** Incorrect workspace path parameter

**Solution:**
1. Verify subscription ID, resource group, workspace name
2. Ensure format matches example above
3. Check for typos in workspace name

### Severity Mapping Error (BadRequest)

**Cause:** Invalid severity value

**Solution:**
The Logic App automatically maps SOCRadar risk levels:
- `INFO` → `Informational`
- `LOW` → `Low`
- `MEDIUM` → `Medium`
- `HIGH` → `High`

If you see this error, check SOCRadar is sending valid risk levels.

## Cost Estimate

**Logic App Consumption Plan:**
- Triggers: Per webhook call (varies by SOCRadar alarm volume)
- Actions per call: ~2-4 (depends on alarms in payload)
- Estimated cost: **$1-5/month** (low to moderate alarm volume)

## Security

- Webhook URL includes SAS token for authentication
- API connection uses OAuth for Sentinel access
- HTTPS-only communication
- Secrets encrypted in ARM deployment
- RBAC: Requires Microsoft Sentinel Contributor role

## Related Repositories

**Complete bidirectional sync requires both:**
1. This repository - SOCRadar → Sentinel (instant)
2. [Bidirectional Integration](https://github.com/orcunsami/azure-bidirectional-incident-app-fork-orcun) - Sentinel → SOCRadar (5-min delay)

## FAQ

**Q: What happens if SOCRadar sends duplicate alarms?**
A: Each alarm creates a new incident. Sentinel's built-in deduplication can merge similar incidents.

**Q: Can I customize the incident title format?**
A: Yes, edit the Logic App designer and modify the Create Incident action's title expression.

**Q: Does this work with SOCRadar Cloud and On-Premises?**
A: Yes, as long as SOCRadar can reach the webhook URL (Azure public endpoint).

**Q: How do I disable the webhook?**
A: Disable the Logic App or remove the webhook URL from SOCRadar platform.

## Support

1. Check [Troubleshooting](#troubleshooting) section
2. Review Logic App run history for errors
3. Verify API connection authorization
4. Open GitHub issue with details
