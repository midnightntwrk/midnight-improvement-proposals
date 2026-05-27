# Midnight Improvement Proposals

This repository is the canonical home for Midnight's open development documents. It is where the community proposes, discusses, and records changes to the Midnight protocol, standards, and ecosystem.

**Contributions from anyone are welcome** — developers, node operators, application builders, token holders, and users alike. If you have an idea for improving Midnight, this is the place to bring it.

## What lives here

- **[`mips/`](./mips)** — **Midnight Improvement Proposals**. Concrete, technically-precise proposals to change the Midnight ecosystem. See [`mips/mip-template.md`](./mips/mip-template.md) and [MIP-0001](./mips/mip-0001-mip-process.md) for the full process.
- **[`mps/`](./mps)** — **Midnight Problem Statements**. Solution-agnostic descriptions of friction, gaps, or opportunities in the ecosystem. MPS documents define *what* needs solving; MIPs propose *how*. See [`mps/mps-template.md`](./mps/mps-template.md).
- **[`deployments/`](./deployments)** — Records of contracts and applications deployed on Midnight, contributed by ecosystem teams. This is a temporary process for MNF internal review of the [contract-deploy-rubric](./mps/contract-deployment-rubric.md) -- no community review is required.
- **[`NUMBERS_INDEX.md`](./NUMBERS_INDEX.md)** — Canonical index of assigned MIP and MPS numbers. Maintainers of this repository should reference this index for the next available number

## How to contribute

1. **Open a discussion first** if you'd like early feedback — via the [Discussions tab](../../discussions) of this repository or our [Discord](https://discord.com/invite/midnightnetwork).
2. **Fork this repository** and copy the appropriate template (`mip-template.md` or `mps-template.md`).
3. **Submit a Pull Request** with your draft. Use `xxxx` in place of a number — an editor will assign one prior to merging.
4. **Engage with reviewers.** Discussion happens in the Discussions tab of this repository. MIP Editors will help shepherd your proposal through the [process stages](./mips/mip-0001-mip-process.md): *Proposed → Review → Accepted → Implemented*.

Pull requests should describe the change, not implement it — link to code repositories rather than including implementation code.

### Authorship and AI assistance

Contributors are welcome to use AI tools (such as Claude) to help draft, edit, or refine their proposals. However, **commits must be authored by the responsible human contributor** — do not configure Claude or any other AI assistant as the commit author or co-author. The named author of a MIP or MPS is accountable for its content, regardless of which tools were used to produce it.

### Signed commits

This repository requires **verified commit signatures**. Commits without a verified signature will be labeled `BLOCKED | Unverified` and cannot be merged until every commit on the PR has a verified signature. See GitHub's guide to [signing commits](https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits) to set this up.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) and [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md) before submitting.

## License

All contributions are licensed under the [Apache License 2.0](./LICENSE).
