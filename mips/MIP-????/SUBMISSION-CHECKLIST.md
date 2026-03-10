# MPS Submission Checklist

Use this checklist before submitting your Midnight Problem Statement for review.

## Pre-Submission Review

### Document Structure

- [ ] Created `MPS-????/` directory
- [ ] Main document is `MPS-????/README.md`
- [ ] Supporting files (if any) are in the same directory
- [ ] File names are lowercase with hyphens (e.g., `architecture-diagram.png`)

### YAML Preamble

- [ ] All required fields present: MPS, Title, Category, Status, Authors, Proposed Solutions, Discussions, Created, License
- [ ] MPS number is `?` (will be assigned on merge)
- [ ] Title is succinct and descriptive (not a full sentence)
- [ ] Category matches one of the defined categories
- [ ] Status is `Open` (unless problem is already solved or inactive)
- [ ] Authors include name and email in format: `Name <email@domain.com>`
- [ ] Proposed Solutions is empty list `[]` or contains MIP references
- [ ] Discussions contains at least one link to GitHub discussion, issue, or Discord thread
- [ ] Created date is in ISO 8601 format (YYYY-MM-DD)
- [ ] License is `CC-BY-4.0` or `Apache-2.0`
- [ ] YAML is valid (paste into YAML validator if unsure)

### Abstract Section

- [ ] Abstract is present and ~200 words
- [ ] Summarizes both goals and obstacles
- [ ] Accessible to Midnight developers (not just domain experts)
- [ ] Does not propose solutions (focuses on problem)

### Problem Section

- [ ] Current state is clearly described
- [ ] Limitations or gaps are identified
- [ ] Context for why this matters is provided
- [ ] Explains why existing approaches are insufficient
- [ ] Impact on users/developers/network is stated
- [ ] Technical depth is appropriate (detailed but not overwhelming)

### Use Cases Section

- [ ] At least 2-3 concrete use cases provided
- [ ] Each use case follows the structure: Scenario, Current Approach, Limitations, Desired Outcome
- [ ] Use cases are realistic (not hypothetical edge cases)
- [ ] Use cases clearly demonstrate the problem
- [ ] User perspective is maintained (not implementation-focused)
- [ ] Use cases cover different aspects or stakeholders

### Goals Section

- [ ] Goals are ranked by priority
- [ ] Primary goals are specific and measurable
- [ ] Requirements section covers: Performance, Security, Privacy, Compatibility, Usability
- [ ] Success metrics are quantifiable where possible
- [ ] Non-goals explicitly define scope boundaries
- [ ] Goals are realistic and achievable
- [ ] Goals do not prescribe specific solutions

### Open Questions Section

- [ ] At least 5 meaningful questions provided
- [ ] Questions probe solution design space
- [ ] Questions identify potential trade-offs
- [ ] Questions address edge cases or failure modes
- [ ] Questions are open-ended (not yes/no)
- [ ] Questions do not assume a particular solution approach

### Copyright Section

- [ ] Explicit license statement included at end
- [ ] License matches YAML preamble

### Optional Sections (if included)

- [ ] References are properly cited with links
- [ ] Appendices provide supporting technical detail
- [ ] Acknowledgements credit contributors appropriately

## Content Quality

### Clarity

- [ ] Problem is clearly articulated (reviewers can understand it)
- [ ] Technical jargon is explained or avoided
- [ ] Writing is concise (no unnecessary verbosity)
- [ ] Sections flow logically
- [ ] Code examples (if any) are properly formatted with syntax highlighting

### Completeness

- [ ] All aspects of the problem are covered
- [ ] Related ecosystem components are identified
- [ ] Dependencies or prerequisites are mentioned
- [ ] Failure modes or edge cases are discussed

### Accuracy

- [ ] Technical details are correct
- [ ] Midnight terminology is used correctly (DUST, nullifiers, commitments, etc.)
- [ ] Performance numbers (if cited) are realistic or measured
- [ ] References to existing code are accurate (file paths, line numbers)

### Focus

- [ ] Problem stays within stated scope
- [ ] Does not drift into solution proposals
- [ ] Avoids unnecessary background or historical context
- [ ] Maintains ecosystem-level perspective (not deployment-specific)

## Community Engagement

### Discussion

- [ ] Posted problem to GitHub Discussions or Discord
- [ ] Received feedback from at least 2-3 community members
- [ ] Incorporated feedback into the document
- [ ] Maintainers are aware and have acknowledged the problem

### Alignment

- [ ] Problem is relevant to Midnight ecosystem (not specific to one deployment)
- [ ] No existing MPSs cover the same problem
- [ ] Problem is not already solved in current implementations
- [ ] Goals align with Midnight's privacy-preserving mission

## Formatting and Style

### Markdown

- [ ] Valid Markdown (renders correctly in preview)
- [ ] Headers use proper hierarchy (# → ## → ###)
- [ ] Lists are properly formatted
- [ ] Code blocks use triple backticks with language tags
- [ ] Links are valid and accessible
- [ ] Tables are properly formatted

### Writing Style

- [ ] Professional and objective tone
- [ ] No AI clichés (crucial, vital, delve, foster, etc.)
- [ ] Active voice preferred over passive
- [ ] Specific examples over vague generalities
- [ ] Direct communication (no "I hope this helps" or similar)

### Midnight Conventions

- [ ] Uses Midnight terminology (not EVM terms)
- [ ] References correct component names (midnight-node, midnight-ledger, etc.)
- [ ] Network names are correct (qanet, preview, testnet, not "dev" or "staging")
- [ ] File paths use correct format: `repository/path/file.ext:line`

## Pre-Submission Actions

### Testing

- [ ] Markdown renders correctly in GitHub
- [ ] All links are valid and reachable
- [ ] Code examples (if any) are tested
- [ ] Supporting files load correctly

### Review

- [ ] Self-reviewed the entire document
- [ ] Asked colleague to review
- [ ] Addressed all review comments
- [ ] Spell-checked and grammar-checked

### Submission

- [ ] Committed to Git with conventional commit message
- [ ] Created PR with descriptive title
- [ ] PR description explains the problem at high level
- [ ] Tagged relevant maintainers for review
- [ ] Linked to discussion threads in PR description

## Common Mistakes to Avoid

### Problem Definition

- [ ] NOT proposing solutions instead of describing problems
- [ ] NOT being too vague ("Midnight needs better performance")
- [ ] NOT being too narrow (one user's specific deployment issue)
- [ ] NOT including every possible detail (focus on what matters)

### Use Cases

- [ ] NOT using hypothetical scenarios ("A user might want...")
- [ ] NOT skipping limitations (must explain why current approaches fail)
- [ ] NOT describing implementation ("The code should...") instead of user needs

### Goals

- [ ] NOT setting unmeasurable goals ("Make it better")
- [ ] NOT setting unrealistic goals ("1ms sync time for entire chain")
- [ ] NOT prescribing solutions ("Use technology X")
- [ ] NOT forgetting non-goals (scope creep prevention)

### Open Questions

- [ ] NOT asking yes/no questions
- [ ] NOT asking leading questions that assume solutions
- [ ] NOT asking trivial questions with obvious answers
- [ ] NOT ignoring hard trade-offs

## Ready to Submit?

If you've checked all applicable items above, your MPS is ready for submission!

### Submission Process

1. **Create PR**: `gh pr create --title "MPS-????: [Your Title]"`
2. **Add Labels**: Request labels from maintainers (e.g., `mps`, `indexer`)
3. **Request Reviews**: Tag relevant domain experts
4. **Monitor Discussion**: Respond to questions and feedback
5. **Iterate**: Update MPS based on review comments
6. **Merge**: Once approved, maintainers will assign MPS number and merge

### After Merge

- MPS number will be assigned (e.g., `MPS-0042`)
- Directory renamed from `MPS-????` to `MPS-0042`
- MPS is now official and can be referenced by solution proposals
- You can propose MIPs to solve the problem
- Monitor discussions for community solution proposals

## Questions?

- **GitHub Discussions**: Ask in the Midnight community
- **Discord**: #development channel for MPS questions
- **Documentation**: See `MPS/README.md` for full process details

---

## Attribution

This MPS process is derived from [CIP-9999: Cardano Problem Statements](https://cips.cardano.org/cip/CIP-9999), licensed under CC-BY-4.0. Adapted for the Midnight ecosystem.

## License

This document is licensed under CC-BY-4.0 (Creative Commons Attribution 4.0 International).

**Version**: 1.0.0
**Last Updated**: 2026-03-09
