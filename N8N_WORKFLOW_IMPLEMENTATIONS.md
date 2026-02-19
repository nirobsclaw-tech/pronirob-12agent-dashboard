# n8n Workflow Implementations
**Ready-to-Use Automation Workflows for Ecom Tracking Service**

## Workflow 1: Client Onboarding Automation

### Trigger: New Client Contract Signed
**Purpose**: Automated onboarding process from contract to project kickoff

```json
{
  "name": "Ecom Tracking - Client Onboarding",
  "nodes": [
    {
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "client-onboarding",
        "responseMode": "responseNode"
      },
      "position": [0, 0]
    },
    {
      "name": "Create Client Record",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "append",
        "documentId": "CLIENT_TRACKER_SHEET_ID",
        "sheetName": "Clients",
        "values": ["={{ $json.client_name }}", "={{ $json.email }}", "={{ $json.tier }}", "={{ $now }}", "pending"]
      }
    },
    {
      "name": "Send Access Request Email",
      "type": "n8n-nodes-base.emailSend",
      "parameters": {
        "fromEmail": "pronirob@pronirob.com",
        "toEmail": "={{ $json.email }}",
        "subject": "🎯 Ready to Fix Your Tracking - Access Request",
        "emailType": "html",
        "message": "=Hi {{ $json.client_name }},\n\nWelcome to your Ecom Tracking optimization project! \n\nTo get started, I need access to your platforms:\n\n{{ $json.tier === 'tier1' ? 'TIER 1 ACCESS NEEDED:\n- Meta Business Manager (Admin)\n- Google Analytics 4 (Admin)\n- Google Tag Manager (Admin)' : $json.tier === 'tier2' ? 'TIER 2 ACCESS NEEDED:\n- Meta Business Manager (Admin)\n- Google Ads Account (Admin)\n- Google Analytics 4 (Admin)\n- Google Tag Manager (Admin)\n- Shopify/WooCommerce Admin' : 'TIER 3 ACCESS NEEDED:\n- Meta Business Manager (Admin)\n- Google Ads Account (Admin)\n- Google Analytics 4 (Admin)\n- Google Tag Manager (Admin)\n- Shopify/WooCommerce Admin\n- Server/Database Access (if applicable)' }}\n\nHere's your access guide: [LINK TO ACCESS GUIDE]\n\nTimeline: I'll complete the full audit within 48 hours of receiving access.\n\nQuestions? Reply to this email or WhatsApp +1234567890\n\nBest regards,\nAs Pro Nirob"
      }
    },
    {
      "name": "Schedule Follow-up",
      "type": "n8n-nodes-base.cron",
      "parameters": {
        "triggerTimes": {
          "item": [
            {
              "hour": 10,
              "minute": 0,
              "dayOfMonth": 3
            }
          ]
        }
      }
    },
    {
      "name": "Create Project Folder",
      "type": "n8n-nodes-base.googleDrive",
      "parameters": {
        "operation": "create",
        "name": "={{ $json.client_name }} - {{ $json.tier }} - {{ $now.toFormat('yyyy-MM-dd') }}"
      }
    },
    {
      "name": "Update Client Status",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "update",
        "documentId": "CLIENT_TRACKER_SHEET_ID",
        "sheetName": "Clients",
        "rowNumber": "={{ $('Create Client Record').item.json.row_number }}"
      }
    },
    {
      "name": "Notification to Team",
      "type": "n8n-nodes-base.openClawMessage",
      "parameters": {
        "channel": "telegram",
        "target": "462062777",
        "message": "🆕 New client onboarded: {{ $json.client_name }} ({{ $json.tier }})\n📧 Access request sent\n📁 Project folder created\n\nTimeline: 48-hour audit completion"
      }
    }
  ],
  "connections": {
    "Webhook Trigger": {
      "main": [
        [
          "Create Client Record",
          "Send Access Request Email",
          "Create Project Folder"
        ]
      ]
    },
    "Send Access Request Email": {
      "main": [
        [
          "Schedule Follow-up"
        ]
      ]
    },
    "Create Client Record": {
      "main": [
        [
          "Update Client Status"
        ]
      ]
    },
    "Schedule Follow-up": {
      "main": [
        [
          "Notification to Team"
        ]
      ]
    }
  },
  "settings": {
    "saveManualExecutions": true,
    "errorWorkflow": "error-handler-workflow"
  }
}
```

## Workflow 2: Meta Platform Audit Automation

### Trigger: Access Received
**Purpose**: Automated Meta platform audit and issue detection

```json
{
  "name": "Meta Platform Audit",
  "nodes": [
    {
      "name": "Access Validation Webhook",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "meta-access-received"
      }
    },
    {
      "name": "Get Meta Business Data",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "https://graph.facebook.com/v18.0/me/adaccounts",
        "method": "GET",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth"
      }
    },
    {
      "name": "Audit Pixel Status",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// Meta Pixel Audit Logic\nconst adAccounts = $input.all()[0].json.data;\nconst auditResults = [];\n\nfor (const account of adAccounts) {\n  // Check pixel firing status\n  const pixelCheck = {\n    account_id: account.id,\n    account_name: account.name,\n    pixel_status: 'unknown',\n    issues: [],\n    priority: 'low'\n  };\n  \n  // Simulate pixel audit (replace with actual API calls)\n  const pixelResponse = await $http.request({\n    method: 'GET',\n    url: `https://graph.facebook.com/v18.0/${account.id}/adspixels`,\n    headers: {\n      'Authorization': 'Bearer ' + process.env.META_ACCESS_TOKEN\n    }\n  });\n  \n  if (pixelResponse.data.data.length === 0) {\n    pixelCheck.pixel_status = 'missing';\n    pixelCheck.issues.push('No Meta pixel found');\n    pixelCheck.priority = 'critical';\n  } else {\n    const pixel = pixelResponse.data.data[0];\n    pixelCheck.pixel_id = pixel.id;\n    pixelCheck.pixel_status = 'active';\n    \n    // Check for common issues\n    if (!pixel.events || pixel.events.length < 3) {\n      pixelCheck.issues.push('Insufficient tracking events');\n      pixelCheck.priority = 'high';\n    }\n  }\n  \n  auditResults.push(pixelCheck);\n}\n\nreturn auditResults.map(result => ({\n  json: {\n    ...result,\n    audit_timestamp: new Date().toISOString(),\n    overall_priority: Math.max(...result.issues.map(() => 1), 1)\n  }\n}));"
      }
    },
    {
      "name": "Calculate Priority Score",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// Calculate overall audit priority\nconst auditResults = $input.all();\n\nlet totalIssues = 0;\nlet criticalIssues = 0;\nlet highIssues = 0;\n\nfor (const item of auditResults) {\n  const result = item.json;\n  totalIssues += result.issues.length;\n  \n  if (result.priority === 'critical') {\n    criticalIssues += result.issues.length;\n  } else if (result.priority === 'high') {\n    highIssues += result.issues.length;\n  }\n}\n\nconst priorityScore = (criticalIssues * 10) + (highIssues * 5) + totalIssues;\n\nreturn {\n  json: {\n    total_issues: totalIssues,\n    critical_issues: criticalIssues,\n    high_issues: highIssues,\n    priority_score: priorityScore,\n    recommended_tier: priorityScore > 20 ? 'tier3' : priorityScore > 10 ? 'tier2' : 'tier1',\n    audit_summary: `Found ${criticalIssues} critical and ${highIssues} high-priority issues`,\n    next_action: criticalIssues > 0 ? 'immediate_fix_required' : 'standard_implementation'\n  }\n};"
      }
    },
    {
      "name": "Generate Audit Report",
      "type": "n8n-nodes-base.googleDocs",
      "parameters": {
        "operation": "create",
        "documentId": "TEMPLATE_DOC_ID",
        "title": "={{ $('Access Validation Webhook').item.json.client_name }} - Meta Audit Report"
      }
    },
    {
      "name": "Send Audit Summary",
      "type": "n8n-nodes-base.openClawMessage",
      "parameters": {
        "channel": "telegram",
        "target": "462062777",
        "message": "📊 Meta Audit Complete for {{ $('Access Validation Webhook').item.json.client_name }}\n\n🔴 Critical Issues: {{ $('Calculate Priority Score').item.json.critical_issues }}\n🟡 High Priority: {{ $('Calculate Priority Score').item.json.high_issues }}\n📈 Priority Score: {{ $('Calculate Priority Score').item.json.priority_score }}\n\n📄 Audit report generated\n⏱️  Estimated implementation: {{ $('Calculate Priority Score').item.json.recommended_tier === 'tier1' ? '1-2 hours' : $('Calculate Priority Score').item.json.recommended_tier === 'tier2' ? '4-6 hours' : '8-12 hours' }}"
      }
    }
  ]
}
```

## Workflow 3: Google Analytics Audit Automation

### Trigger: GA4 Access Received
**Purpose**: Comprehensive GA4 audit with issue detection and prioritization

```json
{
  "name": "GA4 Analytics Audit",
  "nodes": [
    {
      "name": "GA4 Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "parameters": {
        "httpMethod": "POST",
        "path": "ga4-access-received"
      }
    },
    {
      "name": "Get GA4 Properties",
      "type": "n8n-nodes-base.googleAnalytics",
      "parameters": {
        "operation": "getProperties"
      }
    },
    {
      "name": "Audit Property Setup",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// GA4 Property Audit Logic\nconst properties = $input.all()[0].json.properties;\nconst auditResults = [];\n\nfor (const property of properties) {\n  const propertyAudit = {\n    property_id: property.id,\n    property_name: property.name,\n    issues: [],\n    score: 100\n  };\n  \n  // Check for essential GA4 setup\n  if (!property.enhanced_measurement) {\n    propertyAudit.issues.push('Enhanced measurement not enabled');\n    propertyAudit.score -= 20;\n  }\n  \n  // Check e-commerce tracking\n  const ecommerceCheck = await $http.request({\n    method: 'GET',\n    url: `https://analyticsdata.googleapis.com/v1beta/properties/${property.id}/eventNames`,\n    headers: {\n      'Authorization': 'Bearer ' + process.env.GA4_ACCESS_TOKEN\n    }\n  });\n  \n  const events = ecommerceCheck.data.eventNames || [];\n  const requiredEvents = ['page_view', 'view_item', 'add_to_cart', 'purchase'];\n  const missingEvents = requiredEvents.filter(event => !events.includes(event));\n  \n  if (missingEvents.length > 0) {\n    propertyAudit.issues.push(`Missing required events: ${missingEvents.join(', ')}`);\n    propertyAudit.score -= (missingEvents.length * 10);\n  }\n  \n  // Check for duplicate tracking\n  if (events.length > 20) {\n    propertyAudit.issues.push('Possible over-tracking (20+ events)');\n    propertyAudit.score -= 15;\n  }\n  \n  auditResults.push(propertyAudit);\n}\n\nreturn auditResults.map(result => ({ json: result }));"
      }
    },
    {
      "name": "Check Attribution Settings",
      "type": "n8n-nodes-base.httpRequest",
      "parameters": {
        "url": "https://analyticsdata.googleapis.com/v1beta/properties/{property_id}/attributionSettings",
        "method": "GET"
      }
    },
    {
      "name": "Generate Recommendations",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// Generate actionable recommendations\nconst auditResults = $input.all();\nconst recommendations = [];\n\nfor (const item of auditResults) {\n  const result = item.json;\n  \n  for (const issue of result.issues) {\n    let recommendation = '';\n    let priority = 'medium';\n    \n    switch (issue) {\n      case 'Enhanced measurement not enabled':\n        recommendation = 'Enable Enhanced Measurement in GA4 property settings for automatic event tracking';\n        priority = 'high';\n        break;\n      case 'Missing required events':\n        recommendation = 'Implement missing e-commerce events using Google Tag Manager or platform integration';\n        priority = 'critical';\n        break;\n      case 'Possible over-tracking':\n        recommendation = 'Audit event tracking to remove unnecessary or duplicate events';\n        priority = 'medium';\n        break;\n    }\n    \n    recommendations.push({\n      property_id: result.property_id,\n      issue: issue,\n      recommendation: recommendation,\n      priority: priority,\n      estimated_effort: priority === 'critical' ? '2-4 hours' : priority === 'high' ? '1-2 hours' : '30 min - 1 hour'\n    });\n  }\n}\n\nreturn recommendations.map(rec => ({ json: rec }));"
      }
    },
    {
      "name": "Create Action Plan",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "append",
        "documentId": "AUDIT_TRACKER_SHEET_ID",
        "sheetName": "GA4_Actions",
        "values": ["={{ $json.property_id }}", "={{ $json.issue }}", "={{ $json.recommendation }}", "={{ $json.priority }}", "={{ $json.estimated_effort }}", "={{ $now }}"]
      }
    }
  ]
}
```

## Workflow 4: Implementation Progress Tracking

### Trigger: Implementation Milestone
**Purpose**: Track implementation progress and automate client updates

```json
{
  "name": "Implementation Progress Tracking",
  "nodes": [
    {
      "name": "Progress Update Webhook",
      "type": "n8n-nodes-base.webhook",
      "parameter": {
        "httpMethod": "POST",
        "path": "implementation-progress"
      }
    },
    {
      "name": "Update Progress Sheet",
      "type": "n8n-nodes-base.googleSheets",
      "parameters": {
        "operation": "update",
        "documentId": "PROJECT_TRACKER_SHEET_ID",
        "sheetName": "Implementation_Progress",
        "rowNumber": "={{ $json.row_number }}"
      }
    },
    {
      "name": "Check Completion Status",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// Check if implementation is complete\nconst progressData = $input.all()[0].json;\nconst completionThreshold = 95; // 95% completion threshold\n\nif (progressData.completion_percentage >= completionThreshold) {\n  return {\n    json: {\n      ...progressData,\n      status: 'ready_for_delivery',\n      next_action: 'generate_final_report',\n      client_notification_required: true\n    }\n  };\n} else {\n  return {\n    json: {\n      ...progressData,\n      status: 'in_progress',\n      next_action: 'continue_implementation',\n      client_notification_required: progressData.completion_percentage % 25 === 0 // Notify every 25%\n    }\n  };\n}"
      }
    },
    {
      "name": "Generate Client Update",
      "type": "n8n-nodes-base.code",
      "parameters": {
        "jsCode": "// Generate personalized client update\nconst progress = $input.all()[0].json;\n\nlet updateMessage = '';\nlet nextMilestone = '';\n\nswitch (progress.current_phase) {\n  case 'access_collection':\n    updateMessage = `✅ Access collection completed for ${progress.client_name}\\n\\n🔄 Next: Starting comprehensive audit of your tracking setup\\n⏱️ Estimated completion: ${progress.estimated_completion}`;\n    nextMilestone = 'audit_complete';\n    break;\n  case 'audit':\n    updateMessage = `📊 Audit complete for ${progress.client_name}!\\n\\n🎯 Found ${progress.issues_found} tracking issues\\n🔴 ${progress.critical_issues} critical fixes needed\\n\\n⏱️ Implementation starting now\\nEstimated completion: ${progress.estimated_completion}`;\n    nextMilestone = 'implementation_complete';\n    break;\n  case 'implementation':\n    updateMessage = `⚙️ Implementation ${progress.completion_percentage}% complete\\n\\n✅ Completed:\\n${progress.completed_tasks.join('\\n')}\\n\\n⏭️ Next: ${progress.next_tasks.join('\\n')}\\n\\n⏱️ Estimated completion: ${progress.estimated_completion}`;\n    nextMilestone = progress.completion_percentage >= 95 ? 'ready_for_delivery' : 'continue_implementation';\n    break;\n  case 'testing':\n    updateMessage = `🧪 Testing and validation in progress\\n\\n✅ Cross-platform validation\\n✅ Attribution accuracy testing\\n✅ Performance optimization\\n\\n📋 Final report being prepared...`;\n    nextMilestone = 'delivered';\n    break;\n}\n\nreturn {\n  json: {\n    client_name: progress.client_name,\n    client_email: progress.client_email,\n    update_message: updateMessage,\n    completion_percentage: progress.completion_percentage,\n    next_milestone: nextMilestone,\n    estimated_completion: progress.estimated_completion\n  }\n};"
      }
    },
    {
      "name": "Send Client Update",
      "type": "n8n-nodes-base.emailSend",
      "parameters": {
        "fromEmail": "updates@pronirob.com",
        "toEmail": "={{ $json.client_email }}",
        "subject": "🔄 Progress Update: Your Ecom Tracking Project",
        "emailType": "html",
        "message": "={{ $json.update_message }}\\n\\n---\\n\\nNeed help or have questions? Reply to this email or WhatsApp +1234567890\\n\\nBest regards,\\nAs Pro Nirob"
      }
    },
    {
      "name": "Final Report Generation",
      "type": "n8n-nodes-base.conditional",
      "parameters": {
        "conditions": {
          "boolean": [\n            {\n              "value1": "={{ $json.next_milestone }}",\n              "operation": "equal",\n              "value2": "ready_for_delivery"\n            }\n          ]\n        }\n      }\n    }\n  ]\n}\n```\n\n## Workflow 5: Client Satisfaction & Retention Automation\n\n### Trigger: Project Completion
**Purpose**: Automated follow-up for satisfaction and retainer conversion\n\n```json\n{\n  "name": "Client Satisfaction & Retention",\n  "nodes": [\n    {\n      "name": "Project Completion Webhook",\n      "type": "n8n-nodes-base.webhook",\n      "parameters": {\n        "httpMethod": "POST",\n        "path": "project-completed"\n      }\n    },\n    {\n      "name": "Send Satisfaction Survey",\n      "type": "n8n-nodes-base.emailSend",\n      "parameters": {\n        "fromEmail": "feedback@pronirob.com",\n        "toEmail": "={{ $json.client_email }}",\n        "subject": "🎯 Your Tracking Project is Complete - Quick Feedback?",\n        "emailType": "html",\n        "message": "Hi {{ $json.client_name }},\\n\\nYour Ecom Tracking optimization project is complete! 🎉\\n\\nI'd love to hear about your experience:\\n\\n🔗 [1-minute satisfaction survey](SURVEY_LINK)\\n\\nThe survey helps me improve my service and may unlock special offers for you.\\n\\nWhat changed:\\n✅ Enhanced conversion tracking\\n✅ Improved attribution accuracy\\n✅ Optimized pixel performance\\n\\nQuestions? Reply to this email!\\n\\nBest regards,\\nAs Pro Nirob"\n      }\n    },\n    {\n      "name": "Schedule Follow-up",\n      "type": "n8n-nodes-base.cron",\n      "parameters": {\n        "triggerTimes": {\n          "item": [\n            {\n              "hour": 10,\n              "minute": 0,\n              "dayOfMonth": 7\n            }\n          ]\n        }\n      }\n    },\n    {\n      "name": "Send Retainer Offer",\n      "type": "n8n-nodes-base.emailSend",\n      "parameters": {\n        "fromEmail": "offers@pronirob.com",\n        "toEmail": "={{ $json.client_email }}",\n        "subject": "🔒 Keep Your Tracking Perfect - Special Retainer Offer",\n        "emailType": "html",\n        "message": "Hi {{ $json.client_name }},\\n\\nIt's been a week since your tracking optimization, and I hope you're seeing better conversion data! 📈\\n\\nTo keep your tracking perfect long-term, I'm offering our Tracking Guard service:\\n\\n🔒 **Monthly Tracking Guard**\\n✅ 24/7 monitoring and alerts\\n✅ Monthly optimization reviews\\n✅ Quick fixes when issues arise\\n✅ Performance reports\\n\\n**Special launch price: 50% off first month**\\n{{ $json.tier === 'tier1' ? 'Starter: $49/month (normally $99)' : $json.tier === 'tier2' ? 'Core: $99/month (normally $199)' : 'Pro: $199/month (normally $399)' }}\\n\\n[Secure your spot here](SIGNUP_LINK)\\n\\nQuestions? Just reply to this email!\\n\\nBest regards,\\nAs Pro Nirob"\n      }\n    }\n  ]\n}\n```\n\n## Error Handling Workflow\n\n```json\n{\n  "name": "Error Handler & Notification",\n  "nodes": [\n    {\n      "name": "Error Trigger",\n      "type": "n8n-nodes-base.errorTrigger"\n    },\n    {\n      "name": "Log Error Details",\n      "type": "n8n-nodes-base.googleSheets",\n      "parameters": {\n        "operation": "append",\n        "documentId": "ERROR_LOG_SHEET_ID",\n        "sheetName": "Errors",\n        "values": ["={{ $json.error.message }}", "={{ $json.workflow.name }}", "={{ $json.node.name }}", "={{ $now }}", "pending"]\n      }\n    },\n    {\n      "name": "Notify Team",\n      "type": "n8n-nodes-base.openClawMessage",\n      "parameters": {\n        "channel": "telegram",\n        "target": "462062777",\n        "message": "🚨 Automation Error Detected\\n\\nWorkflow: {{ $json.workflow.name }}\\nError: {{ $json.error.message }}\\nNode: {{ $json.node.name }}\\nTime: {{ $now }}\\n\\n⚠️ Manual intervention may be required"\n      }\n    },\n    {\n      "name": "Retry Logic",\n      "type": "n8n-nodes-base.if",\n      "parameters": {\n        "conditions": {\n          "string": [\n            {\n              "value1": "={{ $json.error.message }}",\n              "operation": "notContains",\n              "value2": "rate limit"\n            }\n          ]\n        }\n      }\n    }\n  ]\n}\n```\n\n---\n\n## Implementation Instructions\n\n### 1. Environment Setup\n```bash\n# Install n8n via Docker or npm\nnpm install -g n8n\nn8n start\n\n# Set environment variables\nexport META_ACCESS_TOKEN=your_meta_token\nexport GA4_ACCESS_TOKEN=your_ga4_token\nexport EMAIL_CREDENTIALS=your_smtp_config\nexport GOOGLE_API_CREDENTIALS=your_google_creds\n```\n\n### 2. Required Integrations\n- **Google Sheets API**: For data storage and tracking\n- **Google Drive API**: For document management\n- **Google Analytics API**: For audit automation\n- **Meta Business API**: For platform audits\n- **SMTP Email Service**: For client communications\n- **OpenClaw Gateway**: For Telegram notifications\n\n### 3. Webhook Configuration\nConfigure webhooks in your client management system to trigger workflows:\n- `/client-onboarding`: New contract signed\n- `/meta-access-received`: Meta access confirmed\n- `/ga4-access-received`: GA4 access confirmed\n- `/implementation-progress`: Implementation milestone\n- `/project-completed`: Project delivery complete\n\n### 4. Testing & Validation\n1. **Test each workflow individually** with sample data\n2. **Verify all API connections** and permissions\n3. **Test error handling** with intentional failures\n4. **Validate client communication** flows\n5. **Monitor performance** and optimize as needed\n\nThese workflows provide the foundation for automated delivery of the Ecom Tracking service while maintaining quality and client satisfaction.\n