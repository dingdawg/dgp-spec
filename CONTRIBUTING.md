# Contributing to DGP

Thank you for your interest in contributing to the Decentralized Governance Protocol.

---

## RFC Process

All changes to the DGP specification follow a Request for Comments (RFC) process. This applies to any change that affects:

- Message schemas
- State machine transitions or states
- Protocol layer behavior
- Security model
- Compliance mappings

### Steps

1. **Open an issue** describing the problem you want to solve. Use the title prefix `[RFC]`.
2. **Draft a proposal** in the issue using the template below. Allow at least **14 days** for community comment before moving forward.
3. **Open a pull request** referencing the issue. The PR must include:
   - Updated spec text in `DGP-v1.0.md` (or a new versioned file if breaking)
   - Updated or new JSON schemas in `schemas/` if applicable
   - A changelog entry at the bottom of the spec
4. **Address feedback.** At least two approvals from maintainers are required to merge.
5. **Merged RFCs** are tagged with the next spec version.

### RFC Proposal Template

```
## Problem
What breaks, what is missing, or what is unclear in the current spec?

## Proposed Change
Describe the change precisely. Include example messages or schema diffs.

## Backwards Compatibility
Is this a breaking change? If yes, which spec version will it target?

## Alternatives Considered
What other approaches did you evaluate and why did you reject them?
```

### What Does Not Require an RFC

- Typo and grammar fixes
- Clarification of existing normative language (without changing its meaning)
- Adding examples or improving prose

Open a regular PR for these. One maintainer approval is sufficient.

---

## Contributor License Agreement (CLA)

By submitting a pull request to this repository, you agree to the following terms:

1. **Grant of copyright license.** You grant DingDawg and all recipients of this software a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable copyright license to reproduce, prepare derivative works of, publicly display, publicly perform, sublicense, and distribute your contributions and any derivative works.

2. **Grant of patent license.** You grant DingDawg and all recipients a perpetual, worldwide, non-exclusive, no-charge, royalty-free, irrevocable patent license to make, use, sell, import, and transfer your contributions.

3. **You have the right to grant this license.** Your contribution is your original work, or you have sufficient rights to submit it under these terms.

4. **No warranty.** Your contributions are provided "as is" without warranty of any kind.

This CLA is intentionally lightweight. If your organization requires a separate corporate CLA, open an issue and we will work with you.

---

## Code of Conduct

Be direct, be respectful, stay on topic. Contributions are evaluated on technical merit.

---

## Questions

Open an issue with the prefix `[Question]` and a maintainer will respond within 5 business days.
