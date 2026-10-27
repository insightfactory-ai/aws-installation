# Changelog

Versions your security team can point at. The template is immutable once released:
a change means a new version, never an edit to an existing one.

## Unreleased

First release of the access stack as a downloadable, checksummed file, verifiable
against `SHA256SUMS` in this repository.

### What the stack creates

- **`GitHubOidcProvider`**, an identity provider entry so the account trusts tokens
  issued by GitHub Actions. One per account, controlled by `CreateOidcProvider`.

- **`IFTerraform`**, the IAM role Insight Factory's automation assumes. No user, no
  access key, no password. `MaxSessionDuration` is four hours, because a first install
  builds a VPC with a NAT gateway, a database and a Databricks workspace in one run,
  and a session that expires part way through leaves the deployment half applied.

- **`IFTerraformDeployIam`, `IFTerraformDeployNetwork`, `IFTerraformDeployData` and
  `IFTerraformDeployCompute`**, the four managed policies attached to the role. They
  name 378 individual actions between them, each written in full: there is no asterisk
  anywhere in the four documents, so the answer to "can it do X" is a search of a list
  rather than a judgement about what a wildcard covers. Four policies rather than one
  because a single document would exceed the 6,144 character limit AWS places on a
  managed policy; they are split by job and each can be read on its own.

  The action list is derived from the resource types the deployment declares and
  checked against CloudTrail from live installations. A first install may still stop on
  an `AccessDenied` for an API neither source showed. There is no parameter to widen it
  with: tell us what failed and we will issue a corrected version of this template, so
  the change is reviewable and recorded rather than made once in your account and
  forgotten.

- **`IFTerraformBoundary`**, the permissions boundary carried by every IAM role the
  deployment creates. It is the complete answer to "what is the most any Insight
  Factory role in my account can ever do". Readable, and yours to tighten, but tell us
  if you do: a boundary narrower than the platform needs fails a deployment part way
  through, with resources already created.

### How access is bounded

- **The trust policy names one repository on one branch**, matched exactly rather than
  by pattern:
  `repo:insightfactory-ai@76498963/if_sre_github_releases@1307263479:ref:refs/heads/main`,
  where `76498963` is the `insightfactory-ai` organisation and `1307263479` is the
  `if_sre_github_releases` repository. Naming both by id rather than by name is the
  stronger of the two spellings GitHub issues: a name can be given up and claimed by
  someone else, an id cannot, so a repository renamed to ours would still not match.
  Only this spelling is accepted; accepting both would mean accepting a value that can
  change hands.

- **The role can be assumed from four addresses.** Insight Factory's release pipelines
  have static egress, `192.240.244.134` and `192.240.244.165` in the primary region and
  `152.236.40.73` and `152.236.40.227` in failover, and the trust policy names them.
  The condition sits on the role-assumption call, not on the role's permissions, so it
  never interferes with AWS services acting on the role's behalf afterwards. Our CI
  cannot move without a new version of this template and a stack update you perform.

- **Every role and policy the deployment creates is confined to `IamPath`**,
  default `/insightfactory/`. Its `AllowedPattern` requires at least one path segment,
  so a bare `/` is refused at stack-creation time: it would put our roles at your
  account root, where the path scoping could no longer tell ours apart from yours.

- **Scoping is by resource ARN where the deployment's naming makes it exact**, and by
  condition where it does not. IAM roles and policies are scoped to `IamPath`,
  and the control database connection to one named database user. `iam:CreateRole`
  carries the `IFTerraformBoundary` requirement as a condition on the grant itself, so
  a role cannot be created without the ceiling. The policies that bound the role,
  including the boundary, are created at the account root, outside every scope the role
  is granted, so it cannot edit its own ceiling.

- **Secrets Manager is scoped to this account and Region but not to a name prefix.** A
  client may run more than one naming scheme, so a single prefix would not cover
  everything the deployment needs, and a parameter that is right some of the time is
  worse than none.

- **No IAM group and no IAM user.** The deployment creates no group, adds no one to a
  group and attaches nothing to a user, so no group action and neither
  `iam:AttachUserPolicy` nor `iam:DetachUserPolicy` is granted. Your people's access is
  managed policies under `IamPath`, which you attach to your own groups or IAM Identity
  Center permission sets.

- **Logs are readable only under `/ecs/`**, where the deployment's containers write.
  A release reads a failed schema deploy's output there; the rest of the account's logs
  stay out of reach.

### Parameters and outputs

- **Four parameters, all with working defaults**, and each because your account or
  your organisation decides it rather than us. `CreateOidcProvider`, because an AWS
  account holds one entry per identity provider and only your account state can say
  whether something already trusts GitHub Actions. `IamPath`, because IAM naming
  standards are commonly enforced by an SCP, and a stack that cannot honour yours is a
  stack you cannot deploy at all. `PermissionsBoundaryArn`, because an organisation
  that requires a boundary on every IAM principal would otherwise hit a generic SCP
  denial on `CreateRole` that does not say why; it takes an `AllowedPattern`, so a
  value that is not a policy ARN is refused at stack-creation time.
  `InsightFactoryAccess`, `Enabled` by default, because whether our automation may
  deploy right now is yours to decide: `Disabled` turns the role's trust policy into a
  deny without deleting anything, and the platform keeps running.

  Everything else is fixed in the template: the trusted repository and branch, our CI
  addresses, the maximum session length and the policy names. Each has exactly one
  correct answer, so asking for it would only add a way for the deployed stack to
  differ from the reviewed file.

  `PermissionsBoundaryArn` is a boundary *your* organisation places on `IFTerraform`.
  `IFTerraformBoundary` is the ceiling *we* place on the roles `IFTerraform` creates.
  Both can be set and they do not interact.

- **Outputs are `AwsAccountId`, `Region`, `RoleArn`, `OidcProviderArn`, `IamPath`,
  `DeployPolicyArns` and `BoundaryPolicyArn`.** The handover is the first three values
  per account, plus `IamPath` if you changed it from the default. The rest are for you
  to read. `DeployPolicyArns` lists all four deploy policies.

### Services covered

The four deploy policies cover the twenty-one services the platform uses: `ecr`,
`ec2`, `ecs`, `elasticloadbalancing`, `acm`, `s3`, `kms`, `secretsmanager`, `lambda`,
`glue`, `rds`, `elasticache`, `sns`, `sqs`, `scheduler`, `logs`, `cloudwatch`, `xray`,
`bedrock`, `iam`, `sts`, plus the tagging API. Every other AWS service is absent.

`ecs`, `elasticloadbalancing`, `acm` and `ecr` are there because the platform's web
application and agent applications run on ECS behind an ALB with an ACM certificate.
A release pushes the web application's image to the account's own registry, and runs
the control and hierarchy manager databases' schema deploys as tasks, since neither
database has a public endpoint; so the role may also push images, run and describe tasks,
and read container logs. It may list tasks too, because a schema deploy first looks for a
running task of its own task definition and refuses to start a second while one is still
running. `IFTerraformBoundary`
carries `ecr`, `ecs` and `application-autoscaling` because the roles the deployment
creates for those applications push images, register task definitions and scale the
service, and its `iam:CreateServiceLinkedRole` condition allows
`ecs.application-autoscaling.amazonaws.com` and `spot.amazonaws.com`.

`sqs` is there because each environment's task-run events reach the platform's API
through a queue in the environment's own account. `IFTerraformBoundary` carries it
because the function that sends them runs under a role the deployment creates. The queue
actions are confined to queues in the account and region the stack is deployed in.

`iam:CreateServiceLinkedRole` allows `elasticache.amazonaws.com` because the first
cache created in an account also creates ElastiCache's service-linked role, and an
account that has never held a cache does not have it. `secretsmanager:PutResourcePolicy`
and `secretsmanager:DeleteResourcePolicy` are there because the web applications in each
environment's account read secrets held in the Shared account, and a secret read from
another account needs a resource policy naming who may read it. They are confined to
secrets in the account and region the stack is deployed in, as the other secret actions are.

`lambda:InvokeFunction` is there because the shared stack calls its own login function once
during the apply, to give the shared PostgreSQL its login, and the call is the deploy
role's. `ec2:AssociateVpcCidrBlock` and `ec2:DisassociateVpcCidrBlock` add and remove the
second range the Shared VPC carries for staff access. `iam:UpdateRole`,
`iam:UpdateRoleDescription`, `kms:UpdateKeyDescription` and `iam:UntagPolicy` let an
existing deployment change a role's or key's description and a policy's tags in place
rather than fail; they are confined as the other role, key and policy actions are.

`lambda:PublishLayerVersion` and `lambda:GetLayerVersion` are there because a release of
the orchestrator publishes a new version of its dependency layer and attaches it to the
orchestrator and ticker functions, and attaching a layer needs permission to read that
layer version.

`lambda:PublishVersion`, `lambda:GetAlias`, `lambda:CreateAlias` and `lambda:UpdateAlias`
are there because a release of the orchestrator publishes a numbered version of it and
points its `live` alias at that version, creating the alias on the first release. The
ticker calls the orchestrator through that alias, so work already in progress during a
release finishes on the code it started on. Nothing deletes an alias, so
`lambda:DeleteAlias` is not granted.

### The Terraform state role (optional)

- **`insightfactory-terraform-state.yaml`**, a second template for customers who keep
  Insight Factory's Terraform state in their own S3 bucket. Deployed once, in the
  bucket's account. It creates **`IFTerraformState`**, and the GitHub identity provider
  entry only when `CreateOidcProvider` is `Yes`.

- **What the role can reach.** `s3:ListBucket` on the bucket, `s3:GetObject` and
  `s3:PutObject` on its objects, and `s3:DeleteObject` only on `*.tflock`, the lock files
  Terraform removes when it releases a lock. With `KmsKeyArn`, `kms:Decrypt` and
  `kms:GenerateDataKey` on that key, only through S3 and only for this bucket. Nothing
  else. It cannot delete a state file but can overwrite one, as Terraform must, so the
  bucket should keep versioning on.

- **What it trusts.** The same trust policy as `IFTerraform`, condition for condition,
  with the same four-hour maximum session: Terraform holds the session for a whole plan
  or apply and asks for four hours, and a role allowing less refuses it.

### If you deployed a pre-release copy

An account holding an earlier copy of this template needs a stack update before
anything else works. Its trust policy carries a `sub` value that matches nothing GitHub
sends, so the role denies every assumption, and its policies are missing actions the
deployment calls.
