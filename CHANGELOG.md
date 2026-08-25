# Changelog

Versions your security team can point at. The template is immutable once released —
a change means a new version, never an edit to an existing one.

## Unreleased

First publication of the access stack as a downloadable, checksummed file. It was
previously distributed inline in the installation guide, which gave no way to verify
what you received or to say which version you had reviewed.

Added, relative to the version circulated before this repository existed:

- **`IFTerraformBoundary`** — a managed policy created by the stack that caps every IAM
  role the deployment creates. Readable, and yours to tighten.
- **`IamPath`** (default `/insightfactory/`) — every role and group the deployment
  creates is confined to this path.
- **Five guardrail deny statements on `IFTerraform`.** Explicit denies override every
  allow, including `AdministratorAccess`, so they bound the role whatever
  `DeployPolicyArn` is set to: identities only under the path, no role creation without
  the boundary, no removing the boundary, no rewriting the boundary policy, and no
  access to Organizations, billing, CloudTrail, GuardDuty, Security Hub, Config, IAM
  users or access keys.
- **`IFTerraformDeployPolicy`** — the role no longer uses `AdministratorAccess`.
  `DeployPolicyArn` now defaults to empty, meaning the role is attached to a policy
  this stack creates covering the sixteen services the platform uses: `ec2`, `s3`,
  `kms`, `secretsmanager`, `lambda`, `glue`, `rds`, `elasticache`, `sns`, `scheduler`,
  `logs`, `cloudwatch`, `xray`, `bedrock`, `iam`, `sts`, plus the tagging API. Every
  other AWS service is unreachable. `AdministratorAccess` remains available as a
  documented fallback by supplying it to `DeployPolicyArn`, for the case where a first
  install stops on an `AccessDenied` for something the scoped policy missed.
- **`DeployPolicyInEffect`** output, so it is unambiguous which policy ended up
  attached.
- **`BoundaryPolicyName`** parameter, and `BoundaryPolicyArn` / `IamPath` outputs.
  Neither output needs sending back — we derive both — so the handover is still three
  values per account.

- **`MaxSessionDurationSeconds`** (default `14400`, four hours) — the role's
  `MaxSessionDuration` was fixed at one hour. A first install builds a VPC with a NAT
  gateway, a database and a Databricks workspace in a single run and can take longer
  than that, and the session cannot renew itself part way through: one that expires
  leaves the deployment half applied. The deployment now asks for a session matching
  this value, so lowering it means telling us — a request above the role's maximum is
  refused by AWS and no deployment can start.

- **`TrustedSubject` is now pinned by `AllowedPattern`.** It remains a parameter, so
  the trusted repository and branch appear on your stack's Parameters tab and any
  change is recorded in the stack history — but CloudFormation refuses a wildcard, a
  different repository or a different branch at stack-creation time rather than
  accepting it and quietly weakening the role. A genuine change to this value belongs
  in a new version of the template.

Clarified:

- `PermissionsBoundaryArn` is a boundary *your* organisation places on `IFTerraform`.
  `IFTerraformBoundary` is the ceiling *we* place on the roles it creates. Both can be
  set and they do not interact.
- The scoping notes no longer say the deployment "must also be configured" to apply a
  boundary. It now applies one by default.
