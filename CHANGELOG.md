# Changelog

Versions your security team can point at. The template is immutable once released —
a change means a new version, never an edit to an existing one.

## Unreleased

- **The trusted repository and branch are named by id, not by name.** The `sub`
  condition is now
  `repo:insightfactory-ai@76498963/if_sre_github_releases@1307263479:ref:refs/heads/main`,
  where `76498963` is the `insightfactory-ai` organisation and `1307263479` is the
  `if_sre_github_releases` repository. That is the form GitHub issues for this
  repository, and it is the stronger of the two spellings: a name can be given up and
  claimed by someone else, an id cannot, so a repository renamed to ours afterwards
  still would not match. Still one exact match, never a pattern. The name-based
  spelling is not accepted — accepting both would mean accepting a value that can
  change hands.

  **An account that deployed an earlier version needs a stack update.** Until it is
  applied the role denies every assumption, because the value in its trust policy
  matches nothing GitHub sends.

First publication of the access stack as a downloadable, checksummed file. It was
previously distributed inline in the installation guide, which gave no way to verify
what you received or to say which version you had reviewed.

Changed, relative to the version circulated before this repository existed:

- **The stack asks you for almost nothing.** The trusted repository and branch,
  Insight Factory's CI addresses, the maximum session length and the policy names are
  fixed values in the template. Each had exactly one correct answer that we supplied
  anyway, so asking for it only added a way for the deployed stack to differ from the
  reviewed file, without adding a decision anyone was actually making.

  Three parameters remain, all with working defaults, and each because your account or
  your organisation decides it rather than us. `CreateOidcProvider`, because an AWS
  account holds one entry per identity provider and only your account state can say
  whether something here already trusts GitHub Actions. `IamPath`, because IAM naming
  standards are commonly enforced by an SCP, and a stack that cannot honour yours is a
  stack you cannot deploy at all. And `PermissionsBoundaryArn`, for the same reason —
  an organisation that requires a boundary on every IAM principal would otherwise hit a
  generic SCP denial on `CreateRole` that does not say why. It now takes an
  `AllowedPattern`, so a value that is not a policy ARN is refused at stack-creation
  time rather than accepted and applied.

- **The role can only be assumed from four addresses.** Insight Factory's release
  pipelines have static egress — `192.240.244.134` and `192.240.244.165` in the
  primary region, `152.236.40.73` and `152.236.40.227` in failover — and the trust
  policy names them. The condition sits on the role-assumption call, not on the role's
  permissions, so it never interferes with AWS services acting on the role's behalf
  afterwards. Our CI can no longer move without a new version of this template and a
  stack update you perform.

- **The trusted repository and branch are written into the trust policy**, matched
  exactly rather than by pattern, instead of being supplied as a parameter. This is
  the single control separating our automation from every other caller on GitHub, so
  it is not something a stack deployment should be able to vary.

- **`IFTerraformBoundary`** — a new managed policy created by the stack that caps
  every IAM role the deployment creates. It is the complete answer to "what is the
  most any Insight Factory role in my account can ever do". Readable, and yours to
  tighten — but tell us if you do, because a boundary narrower than the platform needs
  fails a deployment part-way through.

- **Every role and group the deployment creates is confined to `IamPath`**, default
  `/insightfactory/`. Its `AllowedPattern` now requires at least one path segment, so a
  bare `/` is refused at stack-creation time — it would have made the guardrail denies
  below apply to every role in the account rather than to ours, which reads like a
  tightening and is the opposite.

- **Five guardrail deny statements on `IFTerraform`.** Explicit denies override every
  allow, so they bound the role whatever policy is attached to it: identities only
  under the path, no role creation without the boundary, no removing the boundary, no
  rewriting the boundary policy, and no access to Organizations, billing, CloudTrail,
  GuardDuty, Security Hub, Config, IAM users or access keys.

- **`IFTerraformDeploy`** — the role no longer uses `AdministratorAccess`. It is
  attached to a policy this stack creates covering the sixteen services the platform
  uses: `ec2`, `s3`, `kms`, `secretsmanager`, `lambda`, `glue`, `rds`, `elasticache`,
  `sns`, `scheduler`, `logs`, `cloudwatch`, `xray`, `bedrock`, `iam`, `sts`, plus the
  tagging API. Every other AWS service is unreachable.

  There is no parameter to widen this with. If a first install stops on an
  `AccessDenied` for something the scoped policy missed, tell us what failed and we
  will issue a corrected version of the template — which keeps the change reviewable
  and recorded, rather than made once in an account and forgotten.

- **`MaxSessionDuration` is four hours**, up from one. A first install builds a VPC
  with a NAT gateway, a database and a Databricks workspace in a single run and can
  take longer than an hour, and the session cannot renew itself part way through: one
  that expires leaves the deployment half applied.

- **Outputs are `AwsAccountId`, `Region`, `RoleArn`, `OidcProviderArn`, `IamPath`,
  `DeployPolicyArn` and `BoundaryPolicyArn`.** The handover is still the first three
  values per account, plus `IamPath` if you changed it from the default. The rest are
  for you to read.

- **Fixed: `Tags` on the managed policies.** `AWS::IAM::ManagedPolicy` has no `Tags`
  property, so CloudFormation rejected the template at create time with
  `Encountered unsupported property Tags`.

Notes for reviewers who saw the earlier draft:

- `PermissionsBoundaryArn` is a boundary *your* organisation places on `IFTerraform`.
  `IFTerraformBoundary` is the ceiling *we* place on the roles `IFTerraform` creates.
  Both can be set and they do not interact.
- The scoping notes no longer say the deployment "must also be configured" to apply a
  boundary. It now applies one by default.
