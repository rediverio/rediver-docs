---
layout: default
title: Workflow Automation
parent: Features
nav_order: 6
---
{% raw %}

# Workflow Automation

## Overview

Workflows enable automated responses to security events. Unlike Pipelines (which orchestrate scan execution), Workflows handle general automation: notifications, ticket creation, assignments, escalations, and triggering pipelines based on conditions.

## Key Concepts

### Workflows vs Pipelines

| Feature | Workflows | Pipelines |
|---------|-----------|-----------|
| **Purpose** | Automate responses to events | Orchestrate scan execution |
| **Triggers** | Events (findings, assets, schedules) | Manual or scheduled |
| **Actions** | Notifications, tickets, assignments | Run security scans |
| **Nodes** | Trigger → Condition → Action → Notification | Step-based scan stages |

### Workflow Structure

A workflow is a directed graph of nodes connected by edges:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Workflow Graph Structure                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│   ┌──────────┐    ┌───────────┐    ┌──────────┐    ┌──────────┐ │
│   │ TRIGGER  │───▶│ CONDITION │───▶│  ACTION  │───▶│ NOTIFY   │ │
│   │          │    │           │    │          │    │          │ │
│   │ Event or │    │ Check     │    │ Execute  │    │ Alert    │ │
│   │ Schedule │    │ criteria  │    │ response │    │ teams    │ │
│   └──────────┘    └───────────┘    └──────────┘    └──────────┘ │
│                         │                                        │
│                         ▼                                        │
│                   ┌──────────┐                                   │
│                   │  ACTION  │                                   │
│                   │ (branch) │                                   │
│                   └──────────┘                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Node Types

### 1. Trigger Nodes

Trigger nodes define when a workflow starts.

| Trigger Type | Description | Configuration |
|--------------|-------------|---------------|
| `manual` | User-initiated execution | None |
| `schedule` | Cron-based schedule | `cron_expression`, `timezone` |
| `finding_created` | New finding detected | `severity_filter`, `tool_filter` |
| `finding_updated` | Finding status changed | `status_filter`, `field_filter` |
| `finding_age` | Finding exceeds age threshold | `age_days`, `severity_filter` |
| `asset_discovered` | New asset found | `asset_type_filter` |
| `scan_completed` | Scan finishes | `scan_type_filter`, `status_filter` |
| `webhook` | External webhook call | `secret`, `path` |

**Example: Schedule Trigger**
```json
{
  "node_key": "daily_check",
  "node_type": "trigger",
  "name": "Daily SLA Check",
  "config": {
    "trigger_type": "schedule",
    "trigger_config": {
      "cron_expression": "0 9 * * 1-5",
      "timezone": "America/New_York"
    }
  }
}
```

**Example: Finding Created Trigger**
```json
{
  "node_key": "critical_finding",
  "node_type": "trigger",
  "name": "Critical Finding Alert",
  "config": {
    "trigger_type": "finding_created",
    "trigger_config": {
      "severity_filter": ["critical", "high"],
      "tool_filter": ["nuclei", "semgrep"]
    }
  }
}
```

### 2. Condition Nodes

Condition nodes evaluate expressions to determine flow.

**Condition Expression Syntax:**
- Uses CEL (Common Expression Language)
- Access context data via `ctx` variable
- Boolean result determines path (true/false edges)

**Example: Severity Check**
```json
{
  "node_key": "check_severity",
  "node_type": "condition",
  "name": "Is Critical?",
  "config": {
    "condition_expr": "ctx.finding.severity == 'critical'"
  }
}
```

**Example: SLA Check**
```json
{
  "node_key": "check_sla",
  "node_type": "condition",
  "name": "SLA Breached?",
  "config": {
    "condition_expr": "ctx.finding.age_days > ctx.finding.sla_days"
  }
}
```

**Available Context Variables:**

| Variable | Description |
|----------|-------------|
| `ctx.finding` | Finding object (if triggered by finding) |
| `ctx.finding.severity` | Finding severity (critical/high/medium/low) |
| `ctx.finding.age_days` | Days since finding created |
| `ctx.finding.status` | Current status (open/confirmed/in_progress/resolved) |
| `ctx.asset` | Asset object (if triggered by asset) |
| `ctx.scan` | Scan object (if triggered by scan) |
| `ctx.trigger` | Trigger metadata |

### 3. Action Nodes

Action nodes execute automated responses.

| Action Type | Description | Configuration |
|-------------|-------------|---------------|
| `assign_user` | Assign to user | `user_id` |
| `assign_team` | Assign to team | `team_id` |
| `update_priority` | Change priority | `priority` |
| `update_status` | Change status | `status` |
| `add_tags` | Add tags | `tags[]` |
| `remove_tags` | Remove tags | `tags[]` |
| `create_ticket` | Create external ticket | `integration_id`, `template` |
| `update_ticket` | Update external ticket | `integration_id`, `fields` |
| `trigger_pipeline` | Start a scan pipeline | `pipeline_id`, `parameters` |
| `trigger_scan` | Start a quick scan | `scan_profile_id`, `target` |
| `http_request` | Call external API | `url`, `method`, `headers`, `body` |
| `run_script` | Execute custom script | `script`, `timeout` |

**Example: Create Jira Ticket**
```json
{
  "node_key": "create_jira",
  "node_type": "action",
  "name": "Create Jira Issue",
  "config": {
    "action_type": "create_ticket",
    "action_config": {
      "integration_id": "int-jira-123",
      "template": {
        "project": "SEC",
        "issue_type": "Bug",
        "summary": "{{finding.title}}",
        "description": "Severity: {{finding.severity}}\nAsset: {{finding.asset_name}}",
        "priority": "{{severity_to_priority(finding.severity)}}"
      }
    }
  }
}
```

**Example: Trigger Pipeline**
```json
{
  "node_key": "rescan",
  "node_type": "action",
  "name": "Trigger Verification Scan",
  "config": {
    "action_type": "trigger_pipeline",
    "action_config": {
      "pipeline_id": "pip-verify-123",
      "parameters": {
        "target_asset_id": "{{ctx.finding.asset_id}}",
        "finding_id": "{{ctx.finding.id}}"
      }
    }
  }
}
```

### 4. Notification Nodes

Notification nodes send alerts to various channels.

| Notification Type | Description | Configuration |
|-------------------|-------------|---------------|
| `slack` | Slack message | `webhook_url`, `channel`, `template` |
| `email` | Email alert | `to`, `subject`, `template` |
| `teams` | Microsoft Teams | `webhook_url`, `template` |
| `webhook` | Generic webhook | `url`, `method`, `headers`, `body` |
| `pagerduty` | PagerDuty incident | `routing_key`, `severity` |

**Example: Slack Notification**
```json
{
  "node_key": "notify_slack",
  "node_type": "notification",
  "name": "Alert Security Team",
  "config": {
    "notification_type": "slack",
    "notification_config": {
      "webhook_url": "{{secrets.slack_webhook}}",
      "channel": "#security-alerts",
      "template": {
        "text": "🚨 *{{finding.severity | upper}}* finding detected",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*{{finding.title}}*\nAsset: {{finding.asset_name}}\nTool: {{finding.tool}}"
            }
          }
        ]
      }
    }
  }
}
```

## Edges and Flow Control

Edges connect nodes and define execution flow:

```json
{
  "edges": [
    {
      "source_node_key": "trigger_1",
      "target_node_key": "condition_1",
      "label": "always"
    },
    {
      "source_node_key": "condition_1",
      "target_node_key": "action_critical",
      "label": "true"
    },
    {
      "source_node_key": "condition_1",
      "target_node_key": "action_normal",
      "label": "false"
    }
  ]
}
```

**Edge Labels:**
- `always` - Always follow this edge
- `true` - Follow when condition evaluates to true
- `false` - Follow when condition evaluates to false

## Complete Workflow Example

**Scenario:** Alert on critical findings, create Jira ticket, assign to security lead.

```json
{
  "name": "Critical Finding Response",
  "description": "Automated response for critical security findings",
  "is_active": true,
  "tags": ["critical", "automated", "production"],
  "nodes": [
    {
      "node_key": "trigger",
      "node_type": "trigger",
      "name": "Critical Finding Created",
      "config": {
        "trigger_type": "finding_created",
        "trigger_config": {
          "severity_filter": ["critical"]
        }
      },
      "ui_position": {"x": 100, "y": 100}
    },
    {
      "node_key": "check_production",
      "node_type": "condition",
      "name": "Is Production Asset?",
      "config": {
        "condition_expr": "ctx.finding.asset.environment == 'production'"
      },
      "ui_position": {"x": 300, "y": 100}
    },
    {
      "node_key": "create_ticket",
      "node_type": "action",
      "name": "Create Jira Ticket",
      "config": {
        "action_type": "create_ticket",
        "action_config": {
          "integration_id": "int-jira-prod",
          "template": {
            "project": "SEC",
            "issue_type": "Bug",
            "priority": "Highest",
            "summary": "[CRITICAL] {{finding.title}}"
          }
        }
      },
      "ui_position": {"x": 500, "y": 50}
    },
    {
      "node_key": "assign_lead",
      "node_type": "action",
      "name": "Assign to Security Lead",
      "config": {
        "action_type": "assign_user",
        "action_config": {
          "user_id": "user-security-lead"
        }
      },
      "ui_position": {"x": 700, "y": 50}
    },
    {
      "node_key": "notify_slack",
      "node_type": "notification",
      "name": "Alert Slack Channel",
      "config": {
        "notification_type": "slack",
        "notification_config": {
          "webhook_url": "{{secrets.slack_security}}",
          "channel": "#security-critical",
          "template": {
            "text": "🚨 CRITICAL: {{finding.title}} on {{finding.asset_name}}"
          }
        }
      },
      "ui_position": {"x": 900, "y": 50}
    },
    {
      "node_key": "log_only",
      "node_type": "notification",
      "name": "Log for Non-Production",
      "config": {
        "notification_type": "webhook",
        "notification_config": {
          "url": "{{config.logging_endpoint}}",
          "method": "POST",
          "body": {"finding_id": "{{finding.id}}", "environment": "non-production"}
        }
      },
      "ui_position": {"x": 500, "y": 200}
    }
  ],
  "edges": [
    {"source_node_key": "trigger", "target_node_key": "check_production"},
    {"source_node_key": "check_production", "target_node_key": "create_ticket", "label": "true"},
    {"source_node_key": "check_production", "target_node_key": "log_only", "label": "false"},
    {"source_node_key": "create_ticket", "target_node_key": "assign_lead"},
    {"source_node_key": "assign_lead", "target_node_key": "notify_slack"}
  ]
}
```

## Workflow Runs

Each workflow execution creates a Run record:

| Field | Description |
|-------|-------------|
| `id` | Unique run identifier |
| `workflow_id` | Parent workflow |
| `trigger_type` | What triggered this run |
| `trigger_data` | Event data that triggered the run |
| `status` | pending, running, completed, failed, cancelled |
| `context` | Data passed through the workflow |
| `total_nodes` | Total nodes to execute |
| `completed_nodes` | Successfully completed nodes |
| `failed_nodes` | Nodes that failed |
| `started_at` | Execution start time |
| `completed_at` | Execution end time |

### Run Statuses

| Status | Description |
|--------|-------------|
| `pending` | Run created, waiting to start |
| `running` | Currently executing |
| `completed` | All nodes executed successfully |
| `failed` | One or more nodes failed |
| `cancelled` | Manually cancelled |

## API Reference

### Workflows

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/workflows` | Create workflow |
| `GET` | `/api/v1/workflows` | List workflows |
| `GET` | `/api/v1/workflows/{id}` | Get workflow |
| `PUT` | `/api/v1/workflows/{id}` | Update workflow |
| `DELETE` | `/api/v1/workflows/{id}` | Delete workflow |
| `POST` | `/api/v1/workflows/{id}/activate` | Activate workflow |
| `POST` | `/api/v1/workflows/{id}/deactivate` | Deactivate workflow |
| `POST` | `/api/v1/workflows/{id}/clone` | Clone workflow |
| `POST` | `/api/v1/workflows/{id}/trigger` | Manually trigger workflow |

### Workflow Runs

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/workflows/{id}/runs` | List runs for workflow |
| `GET` | `/api/v1/workflow-runs/{id}` | Get run details |
| `POST` | `/api/v1/workflow-runs/{id}/cancel` | Cancel running workflow |
| `GET` | `/api/v1/workflow-runs/{id}/nodes` | Get node execution details |

## Permissions

| Action | Admin | Member | Viewer |
|--------|-------|--------|--------|
| View workflows | Yes | Yes | Yes |
| Create workflows | Yes | Yes | No |
| Update workflows | Yes | Yes | No |
| Delete workflows | Yes | No | No |
| Trigger manually | Yes | Yes | No |
| View runs | Yes | Yes | Yes |
| Cancel runs | Yes | Yes | No |

Permission IDs:
- `workflows:read` - View workflows and runs
- `workflows:write` - Create/update workflows
- `workflows:delete` - Delete workflows
- `workflows:execute` - Trigger workflows manually

## Best Practices

### Design

1. **Start Simple**: Begin with simple trigger → action flows, add conditions later
2. **Name Clearly**: Use descriptive names for nodes (e.g., "Check if Critical" not "Condition 1")
3. **Document Purpose**: Add descriptions to workflows and nodes
4. **Use Tags**: Tag workflows for organization (production, test, critical)

### Security

1. **Secrets Management**: Use `{{secrets.key}}` syntax, never hardcode credentials
2. **Input Validation**: Validate webhook inputs before processing
3. **Rate Limiting**: Consider rate limits for actions that call external APIs
4. **Audit Trail**: All runs are logged with trigger data and outcomes

### Performance

1. **Avoid Loops**: Workflows should be DAGs (no cycles)
2. **Limit Depth**: Keep workflow depth reasonable (< 10 nodes deep)
3. **Batch Operations**: Use batch actions where possible
4. **Async Actions**: Long-running actions are executed asynchronously

### Testing

1. **Test Mode**: Create test workflows with `test` tag
2. **Dry Run**: Use manual trigger with test data before enabling
3. **Monitor Runs**: Check run history for failures and performance
4. **Gradual Rollout**: Enable for non-critical assets first

## Common Workflow Patterns

### SLA Escalation

```
Trigger(finding_age) → Condition(sla_breached) → Action(escalate) → Notify(management)
```

### Auto-Remediation

```
Trigger(finding_created) → Condition(auto_fixable) → Action(trigger_pipeline) → Notify(team)
```

### Multi-Channel Alert

```
Trigger(scan_completed) → Condition(has_critical) → [Notify(slack), Notify(email), Notify(pagerduty)]
```

### Ticket Lifecycle

```
Trigger(finding_updated) → Condition(status_resolved) → Action(update_ticket) → Action(remove_assignment)
```

{% endraw %}
