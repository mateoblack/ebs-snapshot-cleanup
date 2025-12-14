# AWS EBS Snapshot Tag Compliance

Discover untagged EBS snapshots, bulk tag them, and enforce tagging policies with SCPs.

**This runbook is both guided learning and production-ready implementation.** Run through it in a sandbox first, then apply to production with confidence.

## Why?

Untagged snapshots = cost allocation nightmares + security blind spots. This runbook helps you:

1. **Find** all EBS snapshots missing required tags
2. **Fix** them with bulk tagging
3. **Enforce** tagging going forward with SCPs

## Quick Start

```bash
# 1. Set up AWS credentials
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="us-east-1"

# 2. Install dependencies
pip install boto3 pandas

# 3. Run the notebook or follow the runbook
jupyter notebook ebs-snapshot-tag-compliance-runbook.ipynb
```

### Lab Mode (For Testing/Demos)

Don't have existing snapshots? The runbook includes a **Lab Setup** section that creates:
- 5 non-compliant snapshots (missing tags)
- 3 compliant snapshots (with tags)

No EC2 instances required—just creates standalone volumes and snapshots them. Costs ~$0.40/month, cleanup included.

## What's Included

| File | Description |
|------|-------------|
| `ebs-snapshot-tag-compliance-runbook.md` | Step-by-step guide with all the code |
| `ebs-snapshot-tag-compliance-runbook.ipynb` | Jupyter notebook version (interactive) |

## Required Tags

By default, this runbook enforces:

- `Environment` - **Required key AND value validation**
  - Accepted values: `dev`, `Dev`, `development`, `Development`, `staging`, `Staging`, `prod`, `Prod`, `production`, `Production`
  - Missing tag = non-compliant
  - Wrong value (e.g., `Environment=garbage`) = non-compliant
- `CostCenter` - **Required key only** (any value accepted)

Edit `REQUIRED_TAGS` and `VALID_ENVIRONMENT_VALUES` in the notebook to customize.

## Required IAM Permissions

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ConfigPermissions",
            "Effect": "Allow",
            "Action": [
                "config:DescribeConfigurationRecorderStatus",
                "config:DescribeConfigurationRecorders",
                "config:PutConfigurationRecorder",
                "config:PutDeliveryChannel",
                "config:StartConfigurationRecorder",
                "config:PutConfigRule",
                "config:StartConfigRulesEvaluation",
                "config:GetComplianceDetailsByConfigRule",
                "config:DeleteConfigRule"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2Permissions",
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeSnapshots",
                "ec2:CreateTags",
                "ec2:CreateVolume",
                "ec2:DeleteVolume",
                "ec2:CreateSnapshot",
                "ec2:DeleteSnapshot",
                "ec2:DescribeVolumes",
                "ec2:DescribeAvailabilityZones"
            ],
            "Resource": "*"
        },
        {
            "Sid": "CloudTrailPermissions",
            "Effect": "Allow",
            "Action": ["cloudtrail:LookupEvents"],
            "Resource": "*"
        },
        {
            "Sid": "IAMPermissions",
            "Effect": "Allow",
            "Action": ["iam:GetRole", "iam:CreateRole", "iam:AttachRolePolicy", "iam:PassRole"],
            "Resource": ["arn:aws:iam::*:role/AWSConfigRole"]
        },
        {
            "Sid": "S3Permissions",
            "Effect": "Allow",
            "Action": ["s3:CreateBucket", "s3:PutBucketPolicy", "s3:HeadBucket"],
            "Resource": ["arn:aws:s3:::aws-config-bucket-*"]
        },
        {
            "Sid": "OrganizationsPermissions",
            "Effect": "Allow",
            "Action": [
                "organizations:DescribeOrganization",
                "organizations:ListRoots",
                "organizations:ListOrganizationalUnitsForParent",
                "organizations:ListPolicies",
                "organizations:CreatePolicy",
                "organizations:AttachPolicy",
                "organizations:DetachPolicy",
                "organizations:DeletePolicy",
                "organizations:ListTargetsForPolicy"
            ],
            "Resource": "*"
        }
    ]
}
```

## Workflow

```
┌─────────────────┐
│  (Optional)     │
│  Lab Setup      │──┐
│  Create test    │  │
│  snapshots      │  │
└─────────────────┘  │
                     ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Setup AWS      │────▶│  Create Config  │────▶│  Find untagged  │
│  Config         │     │  rule           │     │  snapshots      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
        │                                               │
        │  - Create IAM role                            │
        │  - Create S3 bucket                           ▼
        │  - Start recorder                     ┌─────────────────┐
        │                                       │  Bulk tag       │
        │                                       │  snapshots      │
        │                                       └─────────────────┘
        │                                               │
        │                                               ▼
        │                                       ┌─────────────────┐
        │                                       │  Verify         │
        │                                       │  compliance     │
        │                                       └─────────────────┘
        │                                               │
        ▼                                               ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Done! No more  │◀────│  Attach SCP     │◀────│ ⚠️ CRITICAL:    │
│  untagged snaps │     │  to OUs         │     │ Identify all    │
└─────────────────┘     └─────────────────┘     │ snapshot        │
        │                       │               │ workflows       │
        │                       │               └─────────────────┘
        ▼                       │  
┌─────────────────┐             │  - Verify mgmt account
│  (Optional)     │             │  - List OUs
│  Lab Cleanup    │             │  - Create SCP
│  Delete test    │             │  - Attach to OUs
│  snapshots      │
└─────────────────┘
```

## The SCP

Once everything is tagged, this SCP blocks untagged snapshot creation:

```json
{
    "Effect": "Deny",
    "Action": ["ec2:CreateSnapshot", "ec2:CreateSnapshots"],
    "Resource": "arn:aws:ec2:*::snapshot/*",
    "Condition": {
        "Null": {
            "aws:RequestTag/Environment": "true",
            "aws:RequestTag/CostCenter": "true"
        }
    }
}
```

And enforces valid Environment values:

```json
{
    "Effect": "Deny",
    "Action": ["ec2:CreateSnapshot", "ec2:CreateSnapshots"],
    "Resource": "arn:aws:ec2:*::snapshot/*",
    "Condition": {
        "StringNotEquals": {
            "aws:RequestTag/Environment": [
                "dev", "Dev", "development", "Development",
                "staging", "Staging",
                "prod", "Prod", "production", "Production"
            ]
        }
    }
}
```

## ⚠️ Risk Warning

**This runbook can break production workloads if not implemented carefully.**

When the SCP is enabled, ANY workflow that creates snapshots without tags will fail:
- AWS Backup jobs
- Data Lifecycle Manager (DLM) policies
- CI/CD pipelines
- Lambda functions
- Third-party backup tools
- Manual console snapshots

**Before production:** Identify all snapshot-creating workflows, update them to include tags, and test in a sandbox account first.

## Safety Notes

- All destructive operations are **commented out** by default
- Test in a non-prod account first—this is guided learning AND implementation
- The SCP won't break existing snapshots, only blocks new untagged ones
- You can delete the Config rule anytime without side effects
- **SCPs require the management account** - you can't create them from member accounts
- Creating an SCP doesn't enforce it - you must **attach** it to OUs
- **Rollback:** SCP detachment and deletion functions are included in Cleanup section

## Extending This

Want to enforce tags on other resources? Change the scope:

```python
# In the Config rule
'ComplianceResourceTypes': [
    'AWS::EC2::Snapshot',
    'AWS::EC2::Instance',
    'AWS::EC2::Volume',
    'AWS::S3::Bucket'
]
```

## License

Do whatever you want with it.