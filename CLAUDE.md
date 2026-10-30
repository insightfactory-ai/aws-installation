# CLAUDE.md

The CloudFormation templates a client deploys to give Insight Factory access to its AWS accounts; `README.md` is the client's guide to them.

## Every change leaves the code better

The goal of every change, however small: files better than before — same structure, clearer logic,
more reliable, no new debt.

- **Trace the reader before editing.** The templates are read by the client's security team as well as by
  CloudFormation: a change to a resource, parameter or action list changes `README.md` and `CHANGELOG.md`
  with it.
- **Follow the nearest precedent, then improve what you touch.** Match the naming, layout, comment
  density and changelog wording of the closest resource or entry; fix small faults you pass in the same
  area, and say so in the PR.
- **Generated files are regenerated with their tool, never hand-edited.** `SHA256SUMS` comes from
  `sha256sum` over the templates and must pass `sha256sum -c SHA256SUMS`.
- **A released template is immutable.** A change is a new version in `CHANGELOG.md`, never an edit to a
  released one.
- **Interim values are debt.** A value keyed to one account, id or name is acceptable only to unblock a
  live install, with a tracked removal item and the merge order of the proper fix in the PR body.
- **Prove no regression on an existing installation.** Update the stack in an account that already runs
  the current version, through a change set, and record the result.
- **Changes that other repositories depend on state their merge and deploy order** in the PR body — the
  stack update before the deployment that needs the new permission.
- **Answer every review before handing over**, and record what was run and its result.
