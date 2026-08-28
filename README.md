# Insight Factory on AWS — installation files

Setup files for deploying Insight Factory into your own AWS accounts.

Your team deploys one CloudFormation stack per AWS account. It creates an identity
provider entry and one IAM role, and it is the only thing Insight Factory needs from
you. **No password, access key, or AWS credential of any kind is shared with us.**

Everything in this repository is public so that your cloud and security teams can read
it, check it against the checksums below, and keep a copy of exactly what they
reviewed — without needing an account or asking us for access.

---

## What you deploy

`insightfactory-aws-access.yaml` — about 460 lines, no external dependencies. It
creates four things:

| Resource | What it is |
|---|---|
| `GitHubOidcProvider` | An identity provider entry telling your account to trust tokens issued by GitHub Actions. Created once per account. |
| `IFTerraform` (IAM role) | The role our automation assumes to build and maintain the platform. No user, no key, no password. |
| `IFTerraformDeploy` (managed policy) | The services the role may reach — sixteen of them, and nothing else in AWS. |
| `IFTerraformBoundary` (managed policy) | The ceiling on every IAM role the deployment later creates. Yours to read, and yours to tighten. |

**In the normal case there is nothing to fill in.** Everything the stack trusts — the
repository and branch, our CI addresses, the session length, the policy names — is
written into the template rather than asked of you, so none of it can drift between
the file your security team reviewed and the stack you deployed. Changing any of it
means a new version of the template, not an edit to your copy.

Three parameters remain, all with working defaults, because your account and your
organisation decide them rather than us: whether to create the identity provider
entry, the IAM path, and whether to place one of your own permissions boundaries on
the role. See [Parameters](#parameters).

## How access works

Rather than holding credentials for your account, our automation presents a
short-lived, cryptographically signed token that GitHub issues each time it runs, and
your account exchanges that token for a temporary session in the role above.

The role's trust policy accepts a token only if both claims match exactly:

- `aud` is `sts.amazonaws.com`, so a token minted for any other service cannot be
  replayed here.
- `sub` is one repository on one branch, matched exactly rather than by pattern. No
  other repository on GitHub — including our own — can obtain a session.

  The value names the organisation and the repository by id — `76498963` and
  `1307263479` — rather than by name, which is the form GitHub issues for ours. An id
  cannot be given up and claimed by someone else the way a name can, so a repository
  renamed to ours afterwards still would not match. The name-based spelling is not
  accepted.

And only from four addresses. Our release pipelines have static egress — two in the
primary region, two in the failover region — and the trust policy names them, so the
role cannot be assumed from anywhere else even if something were mis-scoped on our
side. The addresses identify *an Insight Factory CI job*; the repository and branch
identify *the pipeline*. They are independent layers and neither replaces the other.
If our addresses ever change, that is a new version of this template and a stack
update for you — which is the point: our CI cannot move without you seeing it.

The condition applies to the role-assumption call only, not to the role's permissions,
so it never interferes with AWS services acting on the role's behalf afterwards. See
scoping note 4 in the template for why.

There is no shared secret in this arrangement, so there is nothing to rotate, nothing
to leak, and nothing to revoke except the role itself.

## What the role can and cannot do

Two independent limits, and the second holds regardless of the first.

**What it can reach.** The role is attached to a policy this stack creates, covering
the sixteen services the platform uses: `ec2`, `s3`, `kms`, `secretsmanager`,
`lambda`, `glue`, `rds`, `elasticache`, `sns`, `scheduler`, `logs`, `cloudwatch`,
`xray`, `bedrock`, `iam`, `sts`, plus the tagging API. Every other AWS service is
absent, so the role cannot reach any of them. Read `IFTerraformDeployPolicy` in the
template for the exact document.

The grants are at service level rather than enumerated action lists. These accounts
hold nothing but the Insight Factory platform, so a narrower list would mean you
approving a change every time the platform called a new API, without meaningfully
changing what is reachable.

`iam` is in that list because the deployment creates and maintains the roughly fifteen
IAM roles the platform's own services run under. Which brings us to the second limit.

**What it cannot do, whatever policy is attached.** The role carries a set of explicit
denies, and in AWS an explicit deny overrides every allow. They hold whatever policy
is attached to the role, now or in any future version of this template:

1. **It can only touch its own identities.** Every role and group the deployment
   creates lives under `IamPath` (default `/insightfactory/`). All role and group
   mutations, and `iam:PassRole`, are denied anywhere else — so the role cannot read,
   alter or delete an identity it did not create, and cannot hand one of your existing
   privileged roles to a function it creates.
2. **Every role it creates must carry the boundary.** `iam:CreateRole` is denied unless
   the new role is created with `IFTerraformBoundary` attached, so no role it makes can
   exceed that ceiling.
3. **It cannot remove the boundary** from a role once set.
4. **It cannot rewrite the boundary policy** — without this the three above would be
   decorative, because the role could simply publish a new version of the policy that
   caps it.
5. **Account-level controls are yours alone.** AWS Organizations, account settings,
   billing, CloudTrail logging, GuardDuty, Security Hub and Config are denied outright.
   So are `iam:CreateUser` and `iam:CreateAccessKey`, which makes "no users, no keys" a
   property your account enforces rather than a claim we make.

`IFTerraformBoundary` is the complete answer to "what is the most any Insight Factory
role in my account can ever do". What prevents privilege escalation is the absence of
`iam` from that policy, not the precision of the entries in it.

**If a deployment stops on an `AccessDenied`.** The scoped policy is derived from the
resource types the deployment declares rather than from a completed installation's
CloudTrail, so on a first install it may prove very slightly incomplete. The failure
mode is loud — an apply that stops part-way, not a silent gap. There is no parameter
to widen it with: tell us what failed and we will issue a corrected version of this
template. That keeps the change reviewable and recorded, rather than made once in your
account and forgotten.

## Before you start

- The AWS accounts that will host the platform. A full installation is one account per
  environment plus one shared account.
- Permission in them to create an IAM role, an IAM identity provider, an IAM managed
  policy and a CloudFormation stack. That normally means an administrator.
- The region decided.

## Verify the download

The checksums are published here, in this repository, rather than in the email that
pointed you at it. Check the file you downloaded against them before you deploy it.

```bash
# macOS
shasum -a 256 -c SHA256SUMS

# Linux
sha256sum -c SHA256SUMS
```

If it does not match, stop and contact us — do not deploy it.

## Deploy

Deploy the stack **once in each account**, and create it **in the region the platform
will run in**, so the stack records the right one for us. Either route below works —
they do the same thing.

### Console

1. Open **CloudFormation**, check the region in the top-right corner, and choose
   **Create stack → With new resources**.
2. Choose **Upload a template file** and select `insightfactory-aws-access.yaml`.
3. Name the stack `insightfactory-access` and continue past the parameters as they are,
   unless your organisation's standards require a different `IamPath` or a permissions
   boundary on the role.
4. On the last page, acknowledge that the stack creates IAM resources with custom
   names, and create it.
5. When it reaches `CREATE_COMPLETE`, open the **Outputs** tab.

### CLI

```bash
aws cloudformation deploy \
  --template-file insightfactory-aws-access.yaml \
  --stack-name insightfactory-access \
  --region <your-region> \
  --capabilities CAPABILITY_NAMED_IAM

aws cloudformation describe-stacks \
  --stack-name insightfactory-access \
  --region <your-region> \
  --query 'Stacks[0].Outputs'
```

## What to send us

Three values per account, from the **Outputs** tab. None of them is a secret.

| Output | |
|---|---|
| `AwsAccountId` | The account you deployed into |
| `Region` | The region the stack was created in |
| `RoleArn` | The role our automation will assume |

Send them to **deployments@insightfactory.ai**, and tell us which account is which —
which are the environment accounts and which is the shared one.

**Also send `IamPath` if you changed it from the default** — our deployment must be
configured with the same value, and will be denied by its own guardrails on the first
apply if it is not. **And tell us if you set `PermissionsBoundaryArn`.** The remaining
outputs — `OidcProviderArn`, `DeployPolicyArn` and
`BoundaryPolicyArn` — are for you to read. There is nothing to send for any of them.

## Parameters

Three, and each exists because your account or your organisation decides it rather
than us. All three have working defaults, and the defaults are correct unless your
organisation imposes a standard of its own.

| Parameter | Default | |
|---|---|---|
| `CreateOidcProvider` | `Yes` | Set to `No` only if this account already trusts GitHub Actions from an earlier stack. An account holds one entry per identity provider, and a second attempt fails as a duplicate. |
| `IamPath` | `/insightfactory/` | The path every role and group the deployment creates is confined to. Change it if your IAM naming standard requires — and **tell us**, because our deployment must be configured with the same value or it is denied by its own guardrails on the first apply. A bare `/` is refused: it would make the guardrails apply to every role in your account rather than to ours. |
| `PermissionsBoundaryArn` | *(empty)* | Optional. If your organisation requires a permissions boundary on every IAM principal, supply its ARN and it is applied to `IFTerraform`. **Tell us if you set it** — your boundary intersects with everything the role is granted, so one narrower than the platform needs fails a deployment part-way through, with resources already created. |

> `PermissionsBoundaryArn` is a boundary **your** organisation places on `IFTerraform`.
> `IFTerraformBoundary` is the ceiling **we** place on the roles `IFTerraform` later
> creates. Both can be set, and they do not interact.

Everything else is fixed in the template: the trusted repository and branch, our CI
addresses, the four-hour maximum session, and the `IFTerraformDeploy` and
`IFTerraformBoundary` policy names. Read them in the file — that is now the only place
they can be.

> **If you tighten `IFTerraformBoundary`, tell us.** Loosening it is harmless.
> Tightening it below what the platform needs fails a deployment part-way through,
> with resources already created.

## Revoking access

Delete the stack. Every session our automation could obtain disappears with the role,
and there is no credential that outlives it.

To suspend access without deleting, edit or remove the `sub` condition on the role's
trust policy. That change is yours to make, and we cannot widen it from our side.

## Auditing

Every action our automation takes appears in your CloudTrail under an assumed-role
session. Each deployment is also preceded by a Terraform `plan` — a dry run listing
every resource it would create, changing nothing — which we share with you and which
needs your approval before anything is applied.

## Questions

Your Insight Factory contact, or **deployments@insightfactory.ai**.

Please do not open issues here; this repository is a distribution point, not a support
channel.
