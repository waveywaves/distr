# Feature Proposal: Terraform Agent for Distr

**Status**: Draft
**Author**: Engineering
**Created**: 2026-01-20
**Last Updated**: 2026-01-20

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Problem Statement](#problem-statement)
3. [Goals and Non-Goals](#goals-and-non-goals)
4. [User Stories](#user-stories)
5. [Architecture Overview](#architecture-overview)
6. [Detailed Design](#detailed-design)
   - [Database Schema](#database-schema)
   - [API Specifications](#api-specifications)
   - [Agent Implementation](#agent-implementation)
   - [Hub Implementation](#hub-implementation)
   - [Frontend Requirements](#frontend-requirements)
7. [Security Considerations](#security-considerations)
8. [Implementation Phases](#implementation-phases)
9. [Open Questions](#open-questions)
10. [Future Enhancements](#future-enhancements)
11. [Appendix](#appendix)

---

## Executive Summary

This proposal introduces a **Terraform Agent** for Distr, enabling vendors to distribute infrastructure-as-code configurations to self-managed customer environments. Following Distr's existing agent pattern (Docker and Kubernetes agents), the Terraform agent runs in customer environments, pulls configurations from Hub, and provisions cloud infrastructure while keeping credentials local to the customer.

**Key decisions:**
- Customer-side execution (agent model) - credentials never leave customer environment
- Support for both auto-apply and plan-approval workflows
- Customer-managed state backends with optional Hub visibility
- Reuse existing agent communication patterns (polling, JWT auth, status reporting)

---

## Problem Statement

### Current State

Distr currently supports deploying containerized applications via:
- **Docker Agent**: Docker Compose and Docker Swarm deployments
- **Kubernetes Agent**: Helm chart deployments

However, many software products require **cloud infrastructure provisioning** before application deployment:
- Databases (RDS, Cloud SQL, Azure SQL)
- Networking (VPCs, subnets, security groups)
- Storage (S3 buckets, EBS volumes)
- Compute resources (EC2 instances, GKE clusters)
- IAM roles and policies

### Gap

Vendors currently have no way to distribute infrastructure configurations through Distr. Customers must manually provision infrastructure or use separate tooling, leading to:
- Inconsistent deployments across customers
- Manual documentation and support burden
- No visibility into infrastructure state
- Version drift between vendor requirements and customer infrastructure

### Opportunity

Terraform is the industry standard for infrastructure-as-code with:
- Multi-cloud support (AWS, GCP, Azure, and 3000+ providers)
- Declarative configuration
- State management and drift detection
- Large ecosystem of modules and providers

---

## Goals and Non-Goals

### Goals

1. **G1**: Enable vendors to distribute Terraform configurations through Distr Hub
2. **G2**: Allow customers to provision infrastructure in their cloud accounts without sharing credentials
3. **G3**: Provide visibility into Terraform plans before applying changes
4. **G4**: Support approval workflows for infrastructure changes
5. **G5**: Report deployment status, logs, and errors back to Hub
6. **G6**: Follow existing Distr agent patterns for consistency
7. **G7**: Support major cloud providers (AWS, GCP, Azure) and common Terraform backends

### Non-Goals

1. **NG1**: Running Terraform from Hub (vendor-side execution) - security risk
2. **NG2**: Building a Terraform Cloud competitor - integrate, don't replace
3. **NG3**: Supporting Terraform Enterprise features (Sentinel, etc.) in Phase 1
4. **NG4**: Managing Terraform provider credentials from Hub
5. **NG5**: Real-time streaming of Terraform output (batch updates are sufficient)
6. **NG6**: Supporting OpenTofu initially (can be added later as it's API-compatible)

---

## User Stories

### Vendor Personas

**US-V1**: As a vendor, I want to upload Terraform configurations to Distr so that my customers can provision required infrastructure.

**US-V2**: As a vendor, I want to define input variables for my Terraform configurations so that customers can customize deployments.

**US-V3**: As a vendor, I want to see the status of infrastructure deployments across all customers so that I can provide support.

**US-V4**: As a vendor, I want to version my Terraform configurations so that customers can upgrade infrastructure incrementally.

**US-V5**: As a vendor, I want to require approval before Terraform applies so that customers review changes to their infrastructure.

### Customer Personas

**US-C1**: As a customer, I want to install a Terraform agent in my environment so that I can receive infrastructure configurations from my vendor.

**US-C2**: As a customer, I want to review Terraform plans before they apply so that I understand what will change in my infrastructure.

**US-C3**: As a customer, I want to use my existing Terraform state backend (S3, GCS) so that state management follows my organization's practices.

**US-C4**: As a customer, I want to configure cloud credentials locally so that my AWS/GCP/Azure credentials are never shared with the vendor.

**US-C5**: As a customer, I want to see logs and errors from Terraform runs so that I can troubleshoot failures.

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              DISTR HUB                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ Terraform       │  │ Deployment      │  │ Plan Review &           │ │
│  │ Configurations  │  │ Management      │  │ Approval Workflow       │ │
│  │ (Artifacts)     │  │                 │  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │ Status &        │  │ Logs            │  │ REST API                │ │
│  │ Metrics         │  │ Storage         │  │ /api/v1/...             │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│                                                                         │
└─────────────────────────────┬───────────────────────────────────────────┘
                              │
                              │ HTTPS (Polling every 5-30s)
                              │ JWT Authentication
                              │
┌─────────────────────────────┴───────────────────────────────────────────┐
│                        CUSTOMER ENVIRONMENT                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     TERRAFORM AGENT                              │   │
│  │                                                                  │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │   │
│  │  │ Config       │  │ Terraform    │  │ Status               │  │   │
│  │  │ Sync         │  │ Executor     │  │ Reporter             │  │   │
│  │  │              │  │              │  │                      │  │   │
│  │  │ • Poll Hub   │  │ • init       │  │ • Plan output        │  │   │
│  │  │ • Download   │  │ • plan       │  │ • Apply status       │  │   │
│  │  │   configs    │  │ • apply      │  │ • Logs               │  │   │
│  │  │ • Manage     │  │ • destroy    │  │ • Errors             │  │   │
│  │  │   workspaces │  │              │  │                      │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘  │   │
│  │                                                                  │   │
│  │  Environment Variables:                                          │   │
│  │  • DISTR_TARGET_ID, DISTR_TARGET_SECRET (agent auth)            │   │
│  │  • AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (cloud creds)       │   │
│  │  • TF_VAR_* (Terraform variables)                               │   │
│  │                                                                  │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                 │                                       │
│         ┌───────────────────────┼───────────────────────┐               │
│         │                       │                       │               │
│         ▼                       ▼                       ▼               │
│  ┌─────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │ Cloud       │      │ State Backend   │      │ Other           │     │
│  │ Provider    │      │                 │      │ Resources       │     │
│  │             │      │ • S3            │      │                 │     │
│  │ • AWS       │      │ • GCS           │      │ • Databases     │     │
│  │ • GCP       │      │ • Azure Blob    │      │ • Networks      │     │
│  │ • Azure     │      │ • Local         │      │ • Compute       │     │
│  │             │      │ • TF Cloud      │      │                 │     │
│  └─────────────┘      └─────────────────┘      └─────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
┌─────────┐         ┌─────────┐         ┌─────────────┐         ┌───────┐
│ Vendor  │         │   Hub   │         │  TF Agent   │         │ Cloud │
└────┬────┘         └────┬────┘         └──────┬──────┘         └───┬───┘
     │                   │                     │                    │
     │ 1. Upload TF      │                     │                    │
     │    configuration  │                     │                    │
     │──────────────────>│                     │                    │
     │                   │                     │                    │
     │ 2. Create         │                     │                    │
     │    deployment     │                     │                    │
     │──────────────────>│                     │                    │
     │                   │                     │                    │
     │                   │ 3. Poll for         │                    │
     │                   │    resources        │                    │
     │                   │<────────────────────│                    │
     │                   │                     │                    │
     │                   │ 4. Return TF config │                    │
     │                   │────────────────────>│                    │
     │                   │                     │                    │
     │                   │                     │ 5. terraform init  │
     │                   │                     │───────────────────>│
     │                   │                     │                    │
     │                   │                     │ 6. terraform plan  │
     │                   │                     │───────────────────>│
     │                   │                     │<───────────────────│
     │                   │                     │    plan output     │
     │                   │                     │                    │
     │                   │ 7. Report plan      │                    │
     │                   │<────────────────────│                    │
     │                   │                     │                    │
     │ 8. Review plan    │                     │                    │
     │<──────────────────│                     │                    │
     │                   │                     │                    │
     │ 9. Approve        │                     │                    │
     │──────────────────>│                     │                    │
     │                   │                     │                    │
     │                   │ 10. Poll (approved) │                    │
     │                   │<────────────────────│                    │
     │                   │                     │                    │
     │                   │                     │ 11. terraform      │
     │                   │                     │     apply          │
     │                   │                     │───────────────────>│
     │                   │                     │<───────────────────│
     │                   │                     │    resources       │
     │                   │                     │                    │
     │                   │ 12. Report status   │                    │
     │                   │<────────────────────│                    │
     │                   │                     │                    │
     │ 13. View status   │                     │                    │
     │<──────────────────│                     │                    │
     │                   │                     │                    │
```

---

## Detailed Design

### Database Schema

#### New Tables

```sql
-- Terraform configurations (artifact type)
-- Note: Leverages existing artifact system, but with terraform-specific metadata

-- Terraform-specific deployment fields
-- Added to existing deployments table or new terraform_deployments table

CREATE TABLE terraform_deployments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deployment_id UUID NOT NULL REFERENCES deployments(id) ON DELETE CASCADE,

    -- Configuration
    configuration_artifact_id UUID NOT NULL REFERENCES artifacts(id),
    variables JSONB DEFAULT '{}',           -- Input variable values
    backend_config JSONB DEFAULT '{}',      -- Backend configuration overrides
    workspace_name VARCHAR(255) DEFAULT 'default',

    -- Workflow settings
    auto_apply BOOLEAN DEFAULT false,       -- Auto-apply or require approval
    destroy_on_delete BOOLEAN DEFAULT false, -- Run terraform destroy when deployment deleted

    -- Plan management
    current_plan_id UUID REFERENCES terraform_plans(id),

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(deployment_id)
);

-- Terraform plan records
CREATE TABLE terraform_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    terraform_deployment_id UUID NOT NULL REFERENCES terraform_deployments(id) ON DELETE CASCADE,

    -- Plan metadata
    plan_json JSONB,                        -- terraform show -json output
    plan_text TEXT,                         -- Human-readable plan output
    plan_binary BYTEA,                      -- Binary plan file (for apply)

    -- Resource changes summary
    resources_to_add INTEGER DEFAULT 0,
    resources_to_change INTEGER DEFAULT 0,
    resources_to_destroy INTEGER DEFAULT 0,

    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- pending, approved, rejected, applied, failed, expired

    approved_by UUID REFERENCES user_accounts(id),
    approved_at TIMESTAMPTZ,
    rejection_reason TEXT,

    -- Expiration (plans can become stale)
    expires_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    applied_at TIMESTAMPTZ
);

-- Terraform run history
CREATE TABLE terraform_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    terraform_deployment_id UUID NOT NULL REFERENCES terraform_deployments(id) ON DELETE CASCADE,
    plan_id UUID REFERENCES terraform_plans(id),

    -- Run type
    run_type VARCHAR(50) NOT NULL,          -- plan, apply, destroy, refresh

    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    -- pending, running, success, failed, cancelled

    -- Output
    output_log TEXT,
    error_message TEXT,

    -- Timing
    started_at TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Terraform state metadata (optional - for Hub-managed state)
CREATE TABLE terraform_state_metadata (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    terraform_deployment_id UUID NOT NULL REFERENCES terraform_deployments(id) ON DELETE CASCADE,

    -- State info (NOT the actual state, just metadata)
    state_serial INTEGER,
    state_lineage VARCHAR(255),
    resource_count INTEGER,
    last_modified TIMESTAMPTZ,

    -- Outputs (non-sensitive only)
    outputs JSONB DEFAULT '{}',

    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(terraform_deployment_id)
);

-- Indexes
CREATE INDEX idx_terraform_deployments_deployment ON terraform_deployments(deployment_id);
CREATE INDEX idx_terraform_plans_deployment ON terraform_plans(terraform_deployment_id);
CREATE INDEX idx_terraform_plans_status ON terraform_plans(status);
CREATE INDEX idx_terraform_runs_deployment ON terraform_runs(terraform_deployment_id);
CREATE INDEX idx_terraform_runs_status ON terraform_runs(status);
```

#### Modifications to Existing Tables

```sql
-- Add terraform as a deployment type
ALTER TYPE deployment_type ADD VALUE IF NOT EXISTS 'terraform';

-- Add terraform artifact type
ALTER TYPE artifact_type ADD VALUE IF NOT EXISTS 'terraform';

-- deployment_targets table - add terraform-specific fields
ALTER TABLE deployment_targets
ADD COLUMN IF NOT EXISTS terraform_version VARCHAR(50),
ADD COLUMN IF NOT EXISTS terraform_providers JSONB DEFAULT '[]';
-- Tracks which providers/versions are available on the agent
```

### API Specifications

#### Vendor/Admin APIs

##### Upload Terraform Configuration

```
POST /api/v1/artifacts
Content-Type: multipart/form-data

Request:
  - file: terraform-config.tar.gz (required)
    - Contains: *.tf files, modules/, etc.
  - type: "terraform" (required)
  - name: string (required)
  - version: string (required, semver)
  - metadata: JSON (optional)
    {
      "required_providers": {
        "aws": "~> 5.0",
        "kubernetes": "~> 2.0"
      },
      "required_terraform_version": ">= 1.5.0",
      "input_variables": [
        {
          "name": "instance_type",
          "type": "string",
          "description": "EC2 instance type",
          "default": "t3.medium",
          "required": false
        },
        {
          "name": "vpc_cidr",
          "type": "string",
          "description": "VPC CIDR block",
          "required": true
        }
      ],
      "outputs": [
        {
          "name": "cluster_endpoint",
          "description": "Kubernetes cluster endpoint",
          "sensitive": false
        }
      ]
    }

Response: 201 Created
{
  "id": "uuid",
  "name": "my-infrastructure",
  "version": "1.0.0",
  "type": "terraform",
  "metadata": {...},
  "createdAt": "2026-01-20T10:00:00Z"
}
```

##### Create Terraform Deployment

```
POST /api/v1/deployments
Content-Type: application/json

Request:
{
  "name": "customer-xyz-infrastructure",
  "deploymentTargetId": "uuid",
  "type": "terraform",
  "terraform": {
    "artifactId": "uuid",
    "artifactVersion": "1.0.0",
    "variables": {
      "instance_type": "t3.large",
      "vpc_cidr": "10.0.0.0/16",
      "environment": "production"
    },
    "backendConfig": {
      "bucket": "customer-tf-state",
      "key": "infrastructure/terraform.tfstate",
      "region": "us-west-2"
    },
    "workspace": "production",
    "autoApply": false,
    "destroyOnDelete": false
  }
}

Response: 201 Created
{
  "id": "uuid",
  "name": "customer-xyz-infrastructure",
  "type": "terraform",
  "status": "pending",
  "terraform": {
    "artifactId": "uuid",
    "artifactVersion": "1.0.0",
    "workspace": "production",
    "autoApply": false,
    "currentPlan": null
  },
  "createdAt": "2026-01-20T10:00:00Z"
}
```

##### Get Terraform Plan

```
GET /api/v1/deployments/{id}/terraform/plan

Response: 200 OK
{
  "id": "uuid",
  "status": "pending_approval",
  "createdAt": "2026-01-20T10:30:00Z",
  "expiresAt": "2026-01-20T22:30:00Z",
  "summary": {
    "add": 5,
    "change": 2,
    "destroy": 0
  },
  "planText": "Terraform will perform the following actions:\n\n  # aws_instance.web will be created\n  + resource \"aws_instance\" \"web\" {\n      + ami           = \"ami-0c55b159cbfafe1f0\"\n      + instance_type = \"t3.large\"\n      ...\n    }\n\nPlan: 5 to add, 2 to change, 0 to destroy.",
  "resources": [
    {
      "address": "aws_instance.web",
      "action": "create",
      "type": "aws_instance",
      "name": "web",
      "provider": "aws"
    },
    {
      "address": "aws_security_group.web",
      "action": "create",
      "type": "aws_security_group",
      "name": "web",
      "provider": "aws"
    }
  ]
}
```

##### Approve/Reject Plan

```
POST /api/v1/deployments/{id}/terraform/plan/approve
Content-Type: application/json

Request:
{
  "planId": "uuid"
}

Response: 200 OK
{
  "id": "uuid",
  "status": "approved",
  "approvedBy": "user-uuid",
  "approvedAt": "2026-01-20T11:00:00Z"
}
```

```
POST /api/v1/deployments/{id}/terraform/plan/reject
Content-Type: application/json

Request:
{
  "planId": "uuid",
  "reason": "VPC CIDR conflicts with existing network"
}

Response: 200 OK
{
  "id": "uuid",
  "status": "rejected",
  "rejectionReason": "VPC CIDR conflicts with existing network"
}
```

##### Get Terraform Runs

```
GET /api/v1/deployments/{id}/terraform/runs?limit=10&offset=0

Response: 200 OK
{
  "items": [
    {
      "id": "uuid",
      "type": "apply",
      "status": "success",
      "planId": "uuid",
      "startedAt": "2026-01-20T11:00:00Z",
      "completedAt": "2026-01-20T11:05:00Z"
    },
    {
      "id": "uuid",
      "type": "plan",
      "status": "success",
      "startedAt": "2026-01-20T10:30:00Z",
      "completedAt": "2026-01-20T10:31:00Z"
    }
  ],
  "total": 2
}
```

##### Get Terraform State Metadata

```
GET /api/v1/deployments/{id}/terraform/state

Response: 200 OK
{
  "serial": 5,
  "lineage": "abc123",
  "resourceCount": 12,
  "lastModified": "2026-01-20T11:05:00Z",
  "outputs": {
    "cluster_endpoint": {
      "value": "https://k8s.example.com",
      "sensitive": false
    },
    "database_password": {
      "sensitive": true
    }
  }
}
```

##### Trigger Terraform Destroy

```
POST /api/v1/deployments/{id}/terraform/destroy
Content-Type: application/json

Request:
{
  "confirm": true
}

Response: 202 Accepted
{
  "runId": "uuid",
  "status": "pending"
}
```

#### Agent APIs

##### Get Agent Resources (Modified)

```
GET /agent/resources

Response: 200 OK
{
  "agentVersion": "1.0.0",
  "deployments": [
    {
      "id": "uuid",
      "revisionId": "uuid",
      "type": "terraform",
      "terraform": {
        "configurationUrl": "https://hub.distr.sh/api/v1/artifacts/uuid/download",
        "configurationChecksum": "sha256:abc123...",
        "variables": {
          "instance_type": "t3.large",
          "vpc_cidr": "10.0.0.0/16"
        },
        "backendConfig": {
          "bucket": "customer-tf-state",
          "key": "infrastructure/terraform.tfstate",
          "region": "us-west-2"
        },
        "workspace": "production",
        "autoApply": false,
        "pendingAction": "plan",  // plan, apply, destroy, none
        "approvedPlanId": null
      },
      "logsEnabled": true
    }
  ]
}
```

##### Report Terraform Plan

```
POST /agent/terraform/plan
Content-Type: application/json

Request:
{
  "deploymentId": "uuid",
  "revisionId": "uuid",
  "plan": {
    "text": "Terraform will perform...",
    "json": {...},  // terraform show -json output
    "summary": {
      "add": 5,
      "change": 2,
      "destroy": 0
    },
    "resources": [...]
  }
}

Response: 201 Created
{
  "planId": "uuid",
  "status": "pending_approval"
}
```

##### Report Terraform Status

```
POST /agent/status
Content-Type: application/json

Request:
{
  "deploymentId": "uuid",
  "status": "ok",  // ok, progressing, error
  "message": "Terraform apply completed successfully",
  "terraform": {
    "runId": "uuid",
    "runType": "apply",
    "stateMetadata": {
      "serial": 5,
      "lineage": "abc123",
      "resourceCount": 12,
      "outputs": {...}
    }
  }
}

Response: 200 OK
```

##### Report Terraform Logs

```
PUT /agent/logs
Content-Type: application/json

Request:
{
  "records": [
    {
      "deploymentId": "uuid",
      "stream": "terraform",
      "message": "aws_instance.web: Creating...",
      "timestamp": "2026-01-20T11:00:00Z"
    },
    {
      "deploymentId": "uuid",
      "stream": "terraform",
      "message": "aws_instance.web: Creation complete after 30s [id=i-1234567890abcdef0]",
      "timestamp": "2026-01-20T11:00:30Z"
    }
  ]
}

Response: 200 OK
```

### Agent Implementation

#### Directory Structure

```
cmd/agent/terraform/
├── main.go                 # Entry point
├── Dockerfile              # Agent container image
└── README.md

internal/agentterraform/
├── agent.go                # Main agent loop
├── config.go               # Configuration from environment
├── executor.go             # Terraform command execution
├── workspace.go            # Workspace management
├── state.go                # State backend handling
├── plan.go                 # Plan management
├── logs.go                 # Log collection
└── types.go                # Type definitions
```

#### Core Types

```go
// internal/agentterraform/types.go

package agentterraform

import (
    "time"
    "github.com/google/uuid"
)

// TerraformDeployment represents a deployment received from Hub
type TerraformDeployment struct {
    ID                    uuid.UUID              `json:"id"`
    RevisionID            uuid.UUID              `json:"revisionId"`
    ConfigurationURL      string                 `json:"configurationUrl"`
    ConfigurationChecksum string                 `json:"configurationChecksum"`
    Variables             map[string]interface{} `json:"variables"`
    BackendConfig         map[string]string      `json:"backendConfig"`
    Workspace             string                 `json:"workspace"`
    AutoApply             bool                   `json:"autoApply"`
    PendingAction         PendingAction          `json:"pendingAction"`
    ApprovedPlanID        *uuid.UUID             `json:"approvedPlanId"`
    LogsEnabled           bool                   `json:"logsEnabled"`
}

type PendingAction string

const (
    PendingActionNone    PendingAction = "none"
    PendingActionPlan    PendingAction = "plan"
    PendingActionApply   PendingAction = "apply"
    PendingActionDestroy PendingAction = "destroy"
)

// LocalDeployment represents locally tracked deployment state
type LocalDeployment struct {
    ID              uuid.UUID `json:"id"`
    RevisionID      uuid.UUID `json:"revisionId"`
    ConfigChecksum  string    `json:"configChecksum"`
    Workspace       string    `json:"workspace"`
    LastPlanID      *uuid.UUID `json:"lastPlanId"`
    LastAppliedAt   *time.Time `json:"lastAppliedAt"`
    StateSerial     int       `json:"stateSerial"`
}

// PlanResult represents the result of terraform plan
type PlanResult struct {
    Text      string                 `json:"text"`
    JSON      map[string]interface{} `json:"json"`
    Binary    []byte                 `json:"-"`  // Plan file for apply
    Summary   PlanSummary            `json:"summary"`
    Resources []ResourceChange       `json:"resources"`
}

type PlanSummary struct {
    Add     int `json:"add"`
    Change  int `json:"change"`
    Destroy int `json:"destroy"`
}

type ResourceChange struct {
    Address  string `json:"address"`
    Action   string `json:"action"`
    Type     string `json:"type"`
    Name     string `json:"name"`
    Provider string `json:"provider"`
}

// RunResult represents the result of a terraform operation
type RunResult struct {
    Success   bool
    Output    string
    Error     string
    StartedAt time.Time
    EndedAt   time.Time
}
```

#### Agent Configuration

```go
// internal/agentterraform/config.go

package agentterraform

import (
    "os"
    "time"
)

type Config struct {
    // Agent identification
    TargetID     string
    TargetSecret string

    // Hub endpoints
    LoginEndpoint    string
    ResourceEndpoint string
    StatusEndpoint   string
    LogsEndpoint     string
    PlanEndpoint     string

    // Agent settings
    PollInterval      time.Duration
    LogsInterval      time.Duration
    TerraformVersion  string
    WorkDir           string

    // Terraform settings
    TerraformBinary   string
    PluginCacheDir    string

    // Optional Hub-managed state
    UseHubState       bool
    HubStateEndpoint  string
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        TargetID:          os.Getenv("DISTR_TARGET_ID"),
        TargetSecret:      os.Getenv("DISTR_TARGET_SECRET"),
        LoginEndpoint:     os.Getenv("DISTR_LOGIN_ENDPOINT"),
        ResourceEndpoint:  os.Getenv("DISTR_RESOURCE_ENDPOINT"),
        StatusEndpoint:    os.Getenv("DISTR_STATUS_ENDPOINT"),
        LogsEndpoint:      os.Getenv("DISTR_LOGS_ENDPOINT"),
        PlanEndpoint:      os.Getenv("DISTR_PLAN_ENDPOINT"),
        TerraformBinary:   getEnvOrDefault("DISTR_TERRAFORM_BINARY", "terraform"),
        WorkDir:           getEnvOrDefault("DISTR_WORK_DIR", "/var/lib/distr-agent"),
        PluginCacheDir:    getEnvOrDefault("TF_PLUGIN_CACHE_DIR", "/var/cache/terraform"),
        PollInterval:      getDurationOrDefault("DISTR_POLL_INTERVAL", 5*time.Second),
        LogsInterval:      getDurationOrDefault("DISTR_LOGS_INTERVAL", 30*time.Second),
        UseHubState:       os.Getenv("DISTR_USE_HUB_STATE") == "true",
        HubStateEndpoint:  os.Getenv("DISTR_HUB_STATE_ENDPOINT"),
    }

    return cfg, cfg.Validate()
}
```

#### Main Agent Loop

```go
// internal/agentterraform/agent.go

package agentterraform

import (
    "context"
    "time"

    "go.uber.org/zap"
)

type Agent struct {
    config   *Config
    client   *AgentClient
    executor *TerraformExecutor
    logger   *zap.Logger

    localDeployments map[string]*LocalDeployment
}

func NewAgent(config *Config, logger *zap.Logger) (*Agent, error) {
    client, err := NewAgentClient(config)
    if err != nil {
        return nil, err
    }

    executor := NewTerraformExecutor(config, logger)

    return &Agent{
        config:           config,
        client:           client,
        executor:         executor,
        logger:           logger,
        localDeployments: make(map[string]*LocalDeployment),
    }, nil
}

func (a *Agent) Run(ctx context.Context) error {
    // Load existing local deployments
    if err := a.loadLocalDeployments(); err != nil {
        a.logger.Warn("Failed to load local deployments", zap.Error(err))
    }

    ticker := time.NewTicker(a.config.PollInterval)
    defer ticker.Stop()

    logsTicker := time.NewTicker(a.config.LogsInterval)
    defer logsTicker.Stop()

    for {
        select {
        case <-ctx.Done():
            return ctx.Err()

        case <-ticker.C:
            a.reconcile(ctx)

        case <-logsTicker.C:
            a.uploadLogs(ctx)
        }
    }
}

func (a *Agent) reconcile(ctx context.Context) {
    // 1. Fetch desired state from Hub
    resources, err := a.client.GetResources(ctx)
    if err != nil {
        a.logger.Error("Failed to fetch resources", zap.Error(err))
        return
    }

    // 2. Build set of desired deployment IDs
    desiredIDs := make(map[string]bool)
    for _, dep := range resources.Deployments {
        if dep.Type == "terraform" {
            desiredIDs[dep.ID.String()] = true
        }
    }

    // 3. Remove orphaned deployments
    for id, local := range a.localDeployments {
        if !desiredIDs[id] {
            a.handleOrphanedDeployment(ctx, local)
        }
    }

    // 4. Process each terraform deployment
    for _, dep := range resources.Deployments {
        if dep.Type != "terraform" {
            continue
        }
        a.processDeployment(ctx, dep.Terraform)
    }
}

func (a *Agent) processDeployment(ctx context.Context, dep *TerraformDeployment) {
    logger := a.logger.With(zap.String("deploymentId", dep.ID.String()))

    local := a.localDeployments[dep.ID.String()]

    // Check if configuration changed
    configChanged := local == nil ||
        local.RevisionID != dep.RevisionID ||
        local.ConfigChecksum != dep.ConfigurationChecksum

    if configChanged {
        // Download and extract new configuration
        if err := a.downloadConfiguration(ctx, dep); err != nil {
            a.reportError(ctx, dep.ID, "Failed to download configuration: "+err.Error())
            return
        }

        // Run terraform init
        if err := a.executor.Init(ctx, dep); err != nil {
            a.reportError(ctx, dep.ID, "Terraform init failed: "+err.Error())
            return
        }
    }

    // Handle pending actions
    switch dep.PendingAction {
    case PendingActionPlan:
        a.handlePlan(ctx, dep)

    case PendingActionApply:
        a.handleApply(ctx, dep)

    case PendingActionDestroy:
        a.handleDestroy(ctx, dep)

    case PendingActionNone:
        // No action needed, just report current status
        a.reportStatus(ctx, dep.ID, "ok", "No changes pending")
    }
}

func (a *Agent) handlePlan(ctx context.Context, dep *TerraformDeployment) {
    logger := a.logger.With(zap.String("deploymentId", dep.ID.String()))

    a.reportStatus(ctx, dep.ID, "progressing", "Running terraform plan...")

    result, err := a.executor.Plan(ctx, dep)
    if err != nil {
        a.reportError(ctx, dep.ID, "Terraform plan failed: "+err.Error())
        return
    }

    // Report plan to Hub
    planID, err := a.client.ReportPlan(ctx, dep.ID, dep.RevisionID, result)
    if err != nil {
        logger.Error("Failed to report plan", zap.Error(err))
        return
    }

    // Save plan locally for apply
    a.savePlanFile(dep.ID, planID, result.Binary)

    if dep.AutoApply && result.Summary.Destroy == 0 {
        // Auto-apply if enabled and no destroys
        a.handleApply(ctx, dep)
    } else {
        a.reportStatus(ctx, dep.ID, "ok", "Plan complete, awaiting approval")
    }
}

func (a *Agent) handleApply(ctx context.Context, dep *TerraformDeployment) {
    logger := a.logger.With(zap.String("deploymentId", dep.ID.String()))

    // Verify we have an approved plan
    if dep.ApprovedPlanID == nil && !dep.AutoApply {
        logger.Warn("No approved plan for apply")
        return
    }

    a.reportStatus(ctx, dep.ID, "progressing", "Running terraform apply...")

    result, err := a.executor.Apply(ctx, dep)
    if err != nil {
        a.reportError(ctx, dep.ID, "Terraform apply failed: "+err.Error())
        return
    }

    // Update local state
    a.updateLocalDeployment(dep, result)

    // Report state metadata
    stateMetadata, _ := a.executor.GetStateMetadata(ctx, dep)
    a.reportStatusWithState(ctx, dep.ID, "ok", "Terraform apply complete", stateMetadata)
}

func (a *Agent) handleDestroy(ctx context.Context, dep *TerraformDeployment) {
    a.reportStatus(ctx, dep.ID, "progressing", "Running terraform destroy...")

    result, err := a.executor.Destroy(ctx, dep)
    if err != nil {
        a.reportError(ctx, dep.ID, "Terraform destroy failed: "+err.Error())
        return
    }

    // Remove local deployment tracking
    delete(a.localDeployments, dep.ID.String())
    a.removeLocalDeploymentFiles(dep.ID)

    a.reportStatus(ctx, dep.ID, "ok", "Terraform destroy complete")
}
```

#### Terraform Executor

```go
// internal/agentterraform/executor.go

package agentterraform

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "os"
    "os/exec"
    "path/filepath"

    "go.uber.org/zap"
)

type TerraformExecutor struct {
    config *Config
    logger *zap.Logger
}

func NewTerraformExecutor(config *Config, logger *zap.Logger) *TerraformExecutor {
    return &TerraformExecutor{
        config: config,
        logger: logger,
    }
}

func (e *TerraformExecutor) workspaceDir(depID string) string {
    return filepath.Join(e.config.WorkDir, "deployments", depID)
}

func (e *TerraformExecutor) Init(ctx context.Context, dep *TerraformDeployment) error {
    workDir := e.workspaceDir(dep.ID.String())

    args := []string{"init", "-input=false", "-no-color"}

    // Add backend config
    for key, value := range dep.BackendConfig {
        args = append(args, fmt.Sprintf("-backend-config=%s=%s", key, value))
    }

    return e.runCommand(ctx, workDir, args, nil)
}

func (e *TerraformExecutor) Plan(ctx context.Context, dep *TerraformDeployment) (*PlanResult, error) {
    workDir := e.workspaceDir(dep.ID.String())
    planFile := filepath.Join(workDir, "tfplan")

    args := []string{
        "plan",
        "-input=false",
        "-no-color",
        "-detailed-exitcode",
        "-out=" + planFile,
    }

    // Add variables
    varFile, err := e.writeVarFile(workDir, dep.Variables)
    if err != nil {
        return nil, err
    }
    if varFile != "" {
        args = append(args, "-var-file="+varFile)
    }

    var stdout, stderr bytes.Buffer
    err = e.runCommandWithOutput(ctx, workDir, args, nil, &stdout, &stderr)

    // Exit code 2 means changes present (not an error)
    if err != nil {
        if exitErr, ok := err.(*exec.ExitError); ok && exitErr.ExitCode() == 2 {
            // Changes present, this is expected
        } else {
            return nil, fmt.Errorf("plan failed: %s", stderr.String())
        }
    }

    // Read plan file
    planBinary, err := os.ReadFile(planFile)
    if err != nil {
        return nil, err
    }

    // Get JSON representation
    planJSON, err := e.showPlanJSON(ctx, workDir, planFile)
    if err != nil {
        return nil, err
    }

    return &PlanResult{
        Text:      stdout.String(),
        JSON:      planJSON,
        Binary:    planBinary,
        Summary:   e.extractPlanSummary(planJSON),
        Resources: e.extractResourceChanges(planJSON),
    }, nil
}

func (e *TerraformExecutor) Apply(ctx context.Context, dep *TerraformDeployment) (*RunResult, error) {
    workDir := e.workspaceDir(dep.ID.String())
    planFile := filepath.Join(workDir, "tfplan")

    args := []string{
        "apply",
        "-input=false",
        "-no-color",
        "-auto-approve",
    }

    // Use saved plan file if exists
    if _, err := os.Stat(planFile); err == nil {
        args = append(args, planFile)
    } else {
        // No plan file, add variables
        varFile, err := e.writeVarFile(workDir, dep.Variables)
        if err != nil {
            return nil, err
        }
        if varFile != "" {
            args = append(args, "-var-file="+varFile)
        }
    }

    var stdout, stderr bytes.Buffer
    startedAt := time.Now()
    err := e.runCommandWithOutput(ctx, workDir, args, nil, &stdout, &stderr)
    endedAt := time.Now()

    result := &RunResult{
        Success:   err == nil,
        Output:    stdout.String(),
        StartedAt: startedAt,
        EndedAt:   endedAt,
    }

    if err != nil {
        result.Error = stderr.String()
    }

    // Clean up plan file
    os.Remove(planFile)

    return result, err
}

func (e *TerraformExecutor) Destroy(ctx context.Context, dep *TerraformDeployment) (*RunResult, error) {
    workDir := e.workspaceDir(dep.ID.String())

    args := []string{
        "destroy",
        "-input=false",
        "-no-color",
        "-auto-approve",
    }

    // Add variables
    varFile, err := e.writeVarFile(workDir, dep.Variables)
    if err != nil {
        return nil, err
    }
    if varFile != "" {
        args = append(args, "-var-file="+varFile)
    }

    var stdout, stderr bytes.Buffer
    startedAt := time.Now()
    err = e.runCommandWithOutput(ctx, workDir, args, nil, &stdout, &stderr)
    endedAt := time.Now()

    result := &RunResult{
        Success:   err == nil,
        Output:    stdout.String(),
        StartedAt: startedAt,
        EndedAt:   endedAt,
    }

    if err != nil {
        result.Error = stderr.String()
    }

    return result, err
}

func (e *TerraformExecutor) GetStateMetadata(ctx context.Context, dep *TerraformDeployment) (*StateMetadata, error) {
    workDir := e.workspaceDir(dep.ID.String())

    args := []string{"show", "-json"}

    var stdout bytes.Buffer
    if err := e.runCommandWithOutput(ctx, workDir, args, nil, &stdout, nil); err != nil {
        return nil, err
    }

    var state map[string]interface{}
    if err := json.Unmarshal(stdout.Bytes(), &state); err != nil {
        return nil, err
    }

    return e.extractStateMetadata(state), nil
}

func (e *TerraformExecutor) SelectWorkspace(ctx context.Context, dep *TerraformDeployment) error {
    workDir := e.workspaceDir(dep.ID.String())
    workspace := dep.Workspace
    if workspace == "" {
        workspace = "default"
    }

    // Try to select workspace
    args := []string{"workspace", "select", workspace}
    if err := e.runCommand(ctx, workDir, args, nil); err != nil {
        // Workspace doesn't exist, create it
        args = []string{"workspace", "new", workspace}
        return e.runCommand(ctx, workDir, args, nil)
    }

    return nil
}

func (e *TerraformExecutor) runCommand(ctx context.Context, workDir string, args []string, env []string) error {
    return e.runCommandWithOutput(ctx, workDir, args, env, nil, nil)
}

func (e *TerraformExecutor) runCommandWithOutput(
    ctx context.Context,
    workDir string,
    args []string,
    env []string,
    stdout, stderr *bytes.Buffer,
) error {
    cmd := exec.CommandContext(ctx, e.config.TerraformBinary, args...)
    cmd.Dir = workDir
    cmd.Env = append(os.Environ(), env...)

    if stdout != nil {
        cmd.Stdout = stdout
    }
    if stderr != nil {
        cmd.Stderr = stderr
    }

    e.logger.Debug("Running terraform command",
        zap.String("workDir", workDir),
        zap.Strings("args", args),
    )

    return cmd.Run()
}

func (e *TerraformExecutor) writeVarFile(workDir string, variables map[string]interface{}) (string, error) {
    if len(variables) == 0 {
        return "", nil
    }

    varFile := filepath.Join(workDir, "terraform.tfvars.json")
    data, err := json.Marshal(variables)
    if err != nil {
        return "", err
    }

    if err := os.WriteFile(varFile, data, 0600); err != nil {
        return "", err
    }

    return varFile, nil
}

func (e *TerraformExecutor) showPlanJSON(ctx context.Context, workDir, planFile string) (map[string]interface{}, error) {
    args := []string{"show", "-json", planFile}

    var stdout bytes.Buffer
    if err := e.runCommandWithOutput(ctx, workDir, args, nil, &stdout, nil); err != nil {
        return nil, err
    }

    var result map[string]interface{}
    if err := json.Unmarshal(stdout.Bytes(), &result); err != nil {
        return nil, err
    }

    return result, nil
}

func (e *TerraformExecutor) extractPlanSummary(planJSON map[string]interface{}) PlanSummary {
    summary := PlanSummary{}

    if changes, ok := planJSON["resource_changes"].([]interface{}); ok {
        for _, change := range changes {
            if c, ok := change.(map[string]interface{}); ok {
                if actions, ok := c["change"].(map[string]interface{})["actions"].([]interface{}); ok {
                    for _, action := range actions {
                        switch action {
                        case "create":
                            summary.Add++
                        case "update":
                            summary.Change++
                        case "delete":
                            summary.Destroy++
                        }
                    }
                }
            }
        }
    }

    return summary
}

func (e *TerraformExecutor) extractResourceChanges(planJSON map[string]interface{}) []ResourceChange {
    var changes []ResourceChange

    if resourceChanges, ok := planJSON["resource_changes"].([]interface{}); ok {
        for _, rc := range resourceChanges {
            if c, ok := rc.(map[string]interface{}); ok {
                change := ResourceChange{
                    Address:  getString(c, "address"),
                    Type:     getString(c, "type"),
                    Name:     getString(c, "name"),
                    Provider: getString(c, "provider_name"),
                }

                if changeDetails, ok := c["change"].(map[string]interface{}); ok {
                    if actions, ok := changeDetails["actions"].([]interface{}); ok && len(actions) > 0 {
                        change.Action = actions[0].(string)
                    }
                }

                changes = append(changes, change)
            }
        }
    }

    return changes
}
```

#### Dockerfile

```dockerfile
# cmd/agent/terraform/Dockerfile

FROM hashicorp/terraform:1.7 AS terraform

FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -o /distr-agent-terraform ./cmd/agent/terraform

FROM alpine:3.19

RUN apk add --no-cache ca-certificates git openssh-client

# Copy terraform binary
COPY --from=terraform /bin/terraform /usr/local/bin/terraform

# Copy agent binary
COPY --from=builder /distr-agent-terraform /usr/local/bin/distr-agent-terraform

# Create working directories
RUN mkdir -p /var/lib/distr-agent /var/cache/terraform
ENV TF_PLUGIN_CACHE_DIR=/var/cache/terraform
ENV DISTR_WORK_DIR=/var/lib/distr-agent

# Run as non-root
RUN adduser -D -u 1000 distr
USER distr

ENTRYPOINT ["/usr/local/bin/distr-agent-terraform"]
```

### Hub Implementation

#### New Handler Files

```
internal/handlers/
├── terraform_deployments.go      # Terraform deployment CRUD
├── terraform_plans.go            # Plan management
├── terraform_runs.go             # Run history
└── agent_terraform.go            # Agent-specific endpoints
```

#### Handler Examples

```go
// internal/handlers/terraform_plans.go

package handlers

import (
    "encoding/json"
    "net/http"

    "github.com/glasskube/distr/internal/apierrors"
    "github.com/glasskube/distr/internal/db"
    "github.com/glasskube/distr/internal/types"
    "github.com/go-chi/chi/v5"
    "github.com/google/uuid"
)

type TerraformPlanHandler struct {
    db *db.Queries
}

func NewTerraformPlanHandler(db *db.Queries) *TerraformPlanHandler {
    return &TerraformPlanHandler{db: db}
}

// GET /api/v1/deployments/{id}/terraform/plan
func (h *TerraformPlanHandler) GetCurrentPlan(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    deploymentID, err := uuid.Parse(chi.URLParam(r, "id"))
    if err != nil {
        apierrors.BadRequest("Invalid deployment ID").Render(w)
        return
    }

    plan, err := h.db.GetCurrentTerraformPlan(ctx, deploymentID)
    if err != nil {
        apierrors.HandleDBError(w, err)
        return
    }

    json.NewEncoder(w).Encode(plan)
}

// POST /api/v1/deployments/{id}/terraform/plan/approve
func (h *TerraformPlanHandler) ApprovePlan(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    user := context.GetUser(ctx)

    deploymentID, err := uuid.Parse(chi.URLParam(r, "id"))
    if err != nil {
        apierrors.BadRequest("Invalid deployment ID").Render(w)
        return
    }

    var req struct {
        PlanID uuid.UUID `json:"planId"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        apierrors.BadRequest("Invalid request body").Render(w)
        return
    }

    // Verify plan belongs to deployment and is pending
    plan, err := h.db.GetTerraformPlan(ctx, req.PlanID)
    if err != nil {
        apierrors.HandleDBError(w, err)
        return
    }

    if plan.Status != "pending" {
        apierrors.BadRequest("Plan is not pending approval").Render(w)
        return
    }

    // Approve the plan
    plan, err = h.db.ApproveTerraformPlan(ctx, db.ApproveTerraformPlanParams{
        ID:         req.PlanID,
        ApprovedBy: user.ID,
    })
    if err != nil {
        apierrors.HandleDBError(w, err)
        return
    }

    json.NewEncoder(w).Encode(plan)
}

// POST /api/v1/deployments/{id}/terraform/plan/reject
func (h *TerraformPlanHandler) RejectPlan(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    deploymentID, err := uuid.Parse(chi.URLParam(r, "id"))
    if err != nil {
        apierrors.BadRequest("Invalid deployment ID").Render(w)
        return
    }

    var req struct {
        PlanID uuid.UUID `json:"planId"`
        Reason string    `json:"reason"`
    }
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        apierrors.BadRequest("Invalid request body").Render(w)
        return
    }

    plan, err := h.db.RejectTerraformPlan(ctx, db.RejectTerraformPlanParams{
        ID:              req.PlanID,
        RejectionReason: req.Reason,
    })
    if err != nil {
        apierrors.HandleDBError(w, err)
        return
    }

    json.NewEncoder(w).Encode(plan)
}
```

```go
// internal/handlers/agent_terraform.go

package handlers

import (
    "encoding/json"
    "net/http"

    "github.com/glasskube/distr/internal/agentcontext"
    "github.com/glasskube/distr/internal/db"
    "github.com/glasskube/distr/internal/types"
)

type AgentTerraformHandler struct {
    db *db.Queries
}

func NewAgentTerraformHandler(db *db.Queries) *AgentTerraformHandler {
    return &AgentTerraformHandler{db: db}
}

// POST /agent/terraform/plan
func (h *AgentTerraformHandler) ReportPlan(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    target := agentcontext.GetDeploymentTarget(ctx)

    var req types.AgentPlanReport
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Verify deployment belongs to this target
    tfDep, err := h.db.GetTerraformDeploymentForTarget(ctx, req.DeploymentID, target.ID)
    if err != nil {
        http.Error(w, "Deployment not found", http.StatusNotFound)
        return
    }

    // Create plan record
    plan, err := h.db.CreateTerraformPlan(ctx, db.CreateTerraformPlanParams{
        TerraformDeploymentID: tfDep.ID,
        PlanJSON:              req.Plan.JSON,
        PlanText:              req.Plan.Text,
        ResourcesToAdd:        req.Plan.Summary.Add,
        ResourcesToChange:     req.Plan.Summary.Change,
        ResourcesToDestroy:    req.Plan.Summary.Destroy,
        Status:                "pending",
    })
    if err != nil {
        http.Error(w, "Failed to create plan", http.StatusInternalServerError)
        return
    }

    // Update deployment's current plan
    h.db.UpdateTerraformDeploymentCurrentPlan(ctx, tfDep.ID, plan.ID)

    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "planId": plan.ID,
        "status": plan.Status,
    })
}
```

### Frontend Requirements

#### New Components

```
frontend/ui/src/app/
├── components/
│   └── terraform/
│       ├── terraform-plan-viewer/
│       │   ├── terraform-plan-viewer.component.ts
│       │   └── terraform-plan-viewer.component.html
│       ├── terraform-variables-form/
│       │   ├── terraform-variables-form.component.ts
│       │   └── terraform-variables-form.component.html
│       ├── terraform-run-history/
│       │   ├── terraform-run-history.component.ts
│       │   └── terraform-run-history.component.html
│       └── terraform-state-viewer/
│           ├── terraform-state-viewer.component.ts
│           └── terraform-state-viewer.component.html
├── pages/
│   └── deployments/
│       └── terraform-deployment-detail/
│           ├── terraform-deployment-detail.component.ts
│           └── terraform-deployment-detail.component.html
└── services/
    └── terraform.service.ts
```

#### TypeScript Types

```typescript
// frontend/ui/src/app/types/terraform.ts

export interface TerraformDeployment {
  id: string;
  deploymentId: string;
  artifactId: string;
  artifactVersion: string;
  variables: Record<string, unknown>;
  backendConfig: Record<string, string>;
  workspace: string;
  autoApply: boolean;
  destroyOnDelete: boolean;
  currentPlan?: TerraformPlan;
  createdAt: string;
  updatedAt: string;
}

export interface TerraformPlan {
  id: string;
  status: 'pending' | 'approved' | 'rejected' | 'applied' | 'failed' | 'expired';
  createdAt: string;
  expiresAt?: string;
  summary: PlanSummary;
  planText: string;
  resources: ResourceChange[];
  approvedBy?: string;
  approvedAt?: string;
  rejectionReason?: string;
}

export interface PlanSummary {
  add: number;
  change: number;
  destroy: number;
}

export interface ResourceChange {
  address: string;
  action: 'create' | 'update' | 'delete' | 'no-op' | 'read';
  type: string;
  name: string;
  provider: string;
}

export interface TerraformRun {
  id: string;
  type: 'plan' | 'apply' | 'destroy' | 'refresh';
  status: 'pending' | 'running' | 'success' | 'failed' | 'cancelled';
  planId?: string;
  outputLog?: string;
  errorMessage?: string;
  startedAt?: string;
  completedAt?: string;
  createdAt: string;
}

export interface TerraformStateMetadata {
  serial: number;
  lineage: string;
  resourceCount: number;
  lastModified: string;
  outputs: Record<string, TerraformOutput>;
}

export interface TerraformOutput {
  value?: unknown;
  sensitive: boolean;
}

export interface TerraformVariable {
  name: string;
  type: string;
  description?: string;
  default?: unknown;
  required: boolean;
}

export interface TerraformArtifactMetadata {
  requiredProviders: Record<string, string>;
  requiredTerraformVersion: string;
  inputVariables: TerraformVariable[];
  outputs: Array<{
    name: string;
    description?: string;
    sensitive: boolean;
  }>;
}
```

#### Terraform Service

```typescript
// frontend/ui/src/app/services/terraform.service.ts

import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import {
  TerraformDeployment,
  TerraformPlan,
  TerraformRun,
  TerraformStateMetadata,
} from '../types/terraform';

@Injectable({
  providedIn: 'root',
})
export class TerraformService {
  private readonly baseUrl = '/api/v1';

  constructor(private http: HttpClient) {}

  // Plan management
  getCurrentPlan(deploymentId: string): Observable<TerraformPlan> {
    return this.http.get<TerraformPlan>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/plan`
    );
  }

  approvePlan(deploymentId: string, planId: string): Observable<TerraformPlan> {
    return this.http.post<TerraformPlan>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/plan/approve`,
      { planId }
    );
  }

  rejectPlan(
    deploymentId: string,
    planId: string,
    reason: string
  ): Observable<TerraformPlan> {
    return this.http.post<TerraformPlan>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/plan/reject`,
      { planId, reason }
    );
  }

  // Run history
  getRuns(
    deploymentId: string,
    limit = 10,
    offset = 0
  ): Observable<{ items: TerraformRun[]; total: number }> {
    return this.http.get<{ items: TerraformRun[]; total: number }>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/runs`,
      { params: { limit, offset } }
    );
  }

  // State
  getStateMetadata(deploymentId: string): Observable<TerraformStateMetadata> {
    return this.http.get<TerraformStateMetadata>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/state`
    );
  }

  // Destroy
  triggerDestroy(deploymentId: string): Observable<{ runId: string }> {
    return this.http.post<{ runId: string }>(
      `${this.baseUrl}/deployments/${deploymentId}/terraform/destroy`,
      { confirm: true }
    );
  }
}
```

#### Plan Viewer Component

```typescript
// frontend/ui/src/app/components/terraform/terraform-plan-viewer/terraform-plan-viewer.component.ts

import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';
import { TerraformPlan, ResourceChange } from '../../../types/terraform';

@Component({
  selector: 'app-terraform-plan-viewer',
  imports: [CommonModule],
  templateUrl: './terraform-plan-viewer.component.html',
})
export class TerraformPlanViewerComponent {
  @Input() plan!: TerraformPlan;
  @Input() canApprove = false;

  @Output() approve = new EventEmitter<void>();
  @Output() reject = new EventEmitter<string>();

  showRawPlan = false;
  rejectionReason = '';

  get hasDestroys(): boolean {
    return this.plan.summary.destroy > 0;
  }

  getActionClass(action: string): string {
    switch (action) {
      case 'create':
        return 'text-green-600 bg-green-50';
      case 'update':
        return 'text-yellow-600 bg-yellow-50';
      case 'delete':
        return 'text-red-600 bg-red-50';
      default:
        return 'text-gray-600 bg-gray-50';
    }
  }

  getActionIcon(action: string): string {
    switch (action) {
      case 'create':
        return '+';
      case 'update':
        return '~';
      case 'delete':
        return '-';
      default:
        return ' ';
    }
  }

  onApprove(): void {
    this.approve.emit();
  }

  onReject(): void {
    this.reject.emit(this.rejectionReason);
  }
}
```

```html
<!-- frontend/ui/src/app/components/terraform/terraform-plan-viewer/terraform-plan-viewer.component.html -->

<div class="bg-white rounded-lg shadow">
  <!-- Header -->
  <div class="px-6 py-4 border-b border-gray-200">
    <div class="flex items-center justify-between">
      <h3 class="text-lg font-medium text-gray-900">Terraform Plan</h3>
      <span
        class="px-2 py-1 text-sm rounded"
        [ngClass]="{
          'bg-yellow-100 text-yellow-800': plan.status === 'pending',
          'bg-green-100 text-green-800': plan.status === 'approved',
          'bg-red-100 text-red-800': plan.status === 'rejected',
          'bg-blue-100 text-blue-800': plan.status === 'applied'
        }"
      >
        {{ plan.status | titlecase }}
      </span>
    </div>
    <p class="mt-1 text-sm text-gray-500">
      Created {{ plan.createdAt | date: 'medium' }}
    </p>
  </div>

  <!-- Summary -->
  <div class="px-6 py-4 bg-gray-50 border-b border-gray-200">
    <div class="flex space-x-6">
      <div class="flex items-center">
        <span class="text-green-600 font-mono text-lg">+{{ plan.summary.add }}</span>
        <span class="ml-2 text-sm text-gray-600">to add</span>
      </div>
      <div class="flex items-center">
        <span class="text-yellow-600 font-mono text-lg">~{{ plan.summary.change }}</span>
        <span class="ml-2 text-sm text-gray-600">to change</span>
      </div>
      <div class="flex items-center">
        <span class="text-red-600 font-mono text-lg">-{{ plan.summary.destroy }}</span>
        <span class="ml-2 text-sm text-gray-600">to destroy</span>
      </div>
    </div>
  </div>

  <!-- Warning for destroys -->
  @if (hasDestroys) {
    <div class="px-6 py-3 bg-red-50 border-b border-red-100">
      <div class="flex items-center">
        <svg class="h-5 w-5 text-red-400" fill="currentColor" viewBox="0 0 20 20">
          <path fill-rule="evenodd" d="M8.257 3.099c.765-1.36 2.722-1.36 3.486 0l5.58 9.92c.75 1.334-.213 2.98-1.742 2.98H4.42c-1.53 0-2.493-1.646-1.743-2.98l5.58-9.92zM11 13a1 1 0 11-2 0 1 1 0 012 0zm-1-8a1 1 0 00-1 1v3a1 1 0 002 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/>
        </svg>
        <span class="ml-2 text-sm text-red-700">
          This plan will destroy {{ plan.summary.destroy }} resource(s). Review carefully before approving.
        </span>
      </div>
    </div>
  }

  <!-- Resource changes -->
  <div class="px-6 py-4">
    <h4 class="text-sm font-medium text-gray-900 mb-3">Resource Changes</h4>
    <div class="space-y-2">
      @for (resource of plan.resources; track resource.address) {
        <div class="flex items-center px-3 py-2 rounded" [ngClass]="getActionClass(resource.action)">
          <span class="font-mono text-lg w-6">{{ getActionIcon(resource.action) }}</span>
          <span class="font-mono text-sm">{{ resource.address }}</span>
          <span class="ml-auto text-xs text-gray-500">{{ resource.provider }}</span>
        </div>
      }
    </div>
  </div>

  <!-- Raw plan toggle -->
  <div class="px-6 py-4 border-t border-gray-200">
    <button
      type="button"
      class="text-sm text-blue-600 hover:text-blue-800"
      (click)="showRawPlan = !showRawPlan"
    >
      {{ showRawPlan ? 'Hide' : 'Show' }} raw plan output
    </button>
    @if (showRawPlan) {
      <pre class="mt-3 p-4 bg-gray-900 text-gray-100 rounded-lg overflow-x-auto text-xs">{{ plan.planText }}</pre>
    }
  </div>

  <!-- Actions -->
  @if (canApprove && plan.status === 'pending') {
    <div class="px-6 py-4 border-t border-gray-200 bg-gray-50">
      <div class="flex items-center justify-end space-x-3">
        <div class="flex-1">
          <input
            type="text"
            placeholder="Rejection reason (optional)"
            class="w-full px-3 py-2 border border-gray-300 rounded-md text-sm"
            [(ngModel)]="rejectionReason"
          />
        </div>
        <button
          type="button"
          class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-md hover:bg-gray-50"
          (click)="onReject()"
        >
          Reject
        </button>
        <button
          type="button"
          class="px-4 py-2 text-sm font-medium text-white bg-blue-600 rounded-md hover:bg-blue-700"
          (click)="onApprove()"
        >
          Approve & Apply
        </button>
      </div>
    </div>
  }
</div>
```

---

## Security Considerations

### Critical Security Requirements

1. **Cloud credentials NEVER leave customer environment**
   - Agent authenticates to cloud providers using local credentials
   - Hub never sees, stores, or transmits cloud provider credentials
   - Credentials configured via environment variables, instance profiles, or secrets

2. **State file security**
   - State files may contain sensitive data (passwords, keys)
   - Hub should NOT store full state files (only metadata)
   - Customer controls state backend location and encryption
   - If Hub-managed state is used, encrypt at rest

3. **Plan output sanitization**
   - Plan output may contain sensitive values
   - Agent should sanitize before sending to Hub
   - Or Hub should filter sensitive values before display

4. **Agent authentication**
   - Same JWT-based auth as existing agents
   - Target ID + secret for initial auth
   - Short-lived tokens with automatic refresh

5. **Configuration integrity**
   - Terraform configurations signed/checksummed
   - Agent verifies checksum before applying
   - Prevent tampering in transit

### Security Implementation Checklist

- [ ] Cloud credentials only in agent environment
- [ ] State file encryption (customer-side)
- [ ] Plan output sanitization
- [ ] Configuration checksums
- [ ] Audit logging for all operations
- [ ] Rate limiting on agent endpoints
- [ ] Input validation on all API endpoints
- [ ] RBAC for plan approval

---

## Implementation Phases

### Phase 1: Foundation (MVP)

**Duration estimate**: Not provided (focus on what, not when)

**Scope**:
- Database schema for terraform deployments, plans, runs
- Basic agent that can init/plan/apply
- Hub API for terraform deployments
- Simple frontend for viewing status
- Local or S3 state backend support
- Auto-apply mode only (no approval workflow)

**Deliverables**:
- [ ] Database migrations
- [ ] Agent binary (`distr-agent-terraform`)
- [ ] Agent Docker image
- [ ] Hub API endpoints (CRUD, status)
- [ ] Agent manifest generation
- [ ] Basic deployment UI
- [ ] Documentation

### Phase 2: Plan Approval Workflow

**Scope**:
- Plan submission from agent to Hub
- Plan viewing in frontend
- Approve/reject workflow
- Plan expiration handling
- Notifications for pending plans

**Deliverables**:
- [ ] Plan API endpoints
- [ ] Plan viewer component
- [ ] Approval workflow UI
- [ ] Email/webhook notifications
- [ ] Plan expiration job

### Phase 3: Enhanced Visibility

**Scope**:
- Run history and logs
- State metadata display
- Outputs visualization
- Drift detection (optional)
- Destroy workflow

**Deliverables**:
- [ ] Run history API and UI
- [ ] Log streaming improvements
- [ ] State metadata display
- [ ] Outputs viewer (with sensitive masking)
- [ ] Destroy confirmation workflow

### Phase 4: Advanced Features

**Scope**:
- Workspace management
- Module registry integration
- Cost estimation
- Policy checks (OPA/Sentinel-like)
- Terraform Cloud integration

**Deliverables**:
- [ ] Multi-workspace support
- [ ] Module versioning
- [ ] Cost estimation integration
- [ ] Policy engine
- [ ] TFC backend support

---

## Open Questions

### Requiring Decisions

1. **State backend strategy**
   - Option A: Customer always provides backend config
   - Option B: Hub provides optional managed backend
   - Option C: Both, with customer choice
   - **Recommendation**: Option C, default to customer-provided

2. **Plan binary storage**
   - Plans must be stored for apply (Terraform requirement)
   - Option A: Store on agent filesystem
   - Option B: Store in Hub (encrypted)
   - **Recommendation**: Option A for simplicity

3. **Terraform version management**
   - Option A: Single version baked into agent image
   - Option B: Multiple versions, selectable per deployment
   - Option C: Customer provides terraform binary
   - **Recommendation**: Option A initially, Option B later

4. **Provider plugin caching**
   - Providers can be large (hundreds of MB)
   - Need caching strategy for agent
   - **Recommendation**: Persistent volume for plugin cache

5. **Sensitive variable handling**
   - How to handle sensitive input variables?
   - Option A: Never store, customer sets via env vars
   - Option B: Encrypted storage in Hub
   - **Recommendation**: Option A for security

### Future Considerations

1. Should we support OpenTofu as an alternative?
2. Integration with external secret managers (Vault, AWS Secrets Manager)?
3. Multi-region deployment support?
4. Terraform workspace per customer environment?
5. Integration with existing CI/CD pipelines?

---

## Future Enhancements

### Short-term

- **Workspace management**: Multiple workspaces per deployment
- **Module registry**: Store Terraform modules in Distr's OCI registry
- **Import wizard**: Help customers import existing infrastructure
- **Variable sets**: Reusable variable collections

### Medium-term

- **Drift detection**: Scheduled refreshes to detect infrastructure drift
- **Cost estimation**: Integration with Infracost or similar
- **Dependency graphing**: Visualize resource dependencies
- **Change history**: Track all changes over time

### Long-term

- **Policy engine**: OPA-based policy checks before apply
- **Multi-cloud orchestration**: Coordinate deployments across clouds
- **GitOps integration**: Sync with Git repositories
- **Terraform Cloud/Enterprise integration**: Use as backend

---

## Appendix

### A. Example Terraform Configuration Package

```
my-infrastructure.tar.gz
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── README.md
```

### B. Agent Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DISTR_TARGET_ID` | Yes | Agent target UUID |
| `DISTR_TARGET_SECRET` | Yes | Agent authentication secret |
| `DISTR_LOGIN_ENDPOINT` | Yes | Hub login URL |
| `DISTR_RESOURCE_ENDPOINT` | Yes | Hub resources URL |
| `DISTR_STATUS_ENDPOINT` | Yes | Hub status URL |
| `DISTR_LOGS_ENDPOINT` | Yes | Hub logs URL |
| `DISTR_PLAN_ENDPOINT` | Yes | Hub plan submission URL |
| `DISTR_POLL_INTERVAL` | No | Polling interval (default: 5s) |
| `DISTR_LOGS_INTERVAL` | No | Logs upload interval (default: 30s) |
| `DISTR_WORK_DIR` | No | Agent working directory |
| `TF_PLUGIN_CACHE_DIR` | No | Terraform plugin cache |
| `AWS_*` | Varies | AWS credentials (if using AWS) |
| `GOOGLE_*` | Varies | GCP credentials (if using GCP) |
| `ARM_*` | Varies | Azure credentials (if using Azure) |

### C. State Backend Examples

**S3 Backend**:
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**GCS Backend**:
```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "infrastructure"
  }
}
```

**Azure Backend**:
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "tfstate"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                  = "terraform.tfstate"
  }
}
```

### D. Related Documents

- [Distr Agent Architecture](../architecture/agents.md)
- [Distr API Reference](../api/README.md)
- [Terraform Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)

---

## Changelog

| Date | Author | Change |
|------|--------|--------|
| 2026-01-20 | Engineering | Initial draft |
