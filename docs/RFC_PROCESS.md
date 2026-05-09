# RFC Process

This document describes the Request for Comments (RFC) process used in FLUX to propose, discuss, and decide on significant architectural changes. The process is modeled after the RFC mechanisms used in Rust, Python (PEPs), Kubernetes (KEPs), and Django.

The RFC process exists to slow down decisions that deserve to be slow. Most contributions to FLUX do not need an RFC — they go through the standard pull request flow described in [CONTRIBUTING.md](../CONTRIBUTING.md). The RFC process is reserved for changes whose impact extends beyond a single feature.

---

## Table of Contents

1. [When an RFC is Required](#when-an-rfc-is-required)
2. [When an RFC is Not Required](#when-an-rfc-is-not-required)
3. [The RFC Lifecycle](#the-rfc-lifecycle)
4. [RFC Document Structure](#rfc-document-structure)
5. [Discussion and Decision](#discussion-and-decision)
6. [Implementation](#implementation)
7. [Withdrawing or Amending an RFC](#withdrawing-or-amending-an-rfc)
8. [Template](#template)

---

## When an RFC is Required

An RFC is required for any change that meets at least one of the following criteria:

**Modifies the public API.** Any change to the symbols exported from `flux/__init__.py`, the signatures of public methods, or the names of registered plugins requires an RFC. Renaming `Pipeline.from_config` is an RFC. Adding a new optional argument to it is not.

**Alters the architectural structure.** Changes to the three-phase decomposition, the queue-based communication model, the execution-mode contract (`simple` vs `fast`), or the plugin registration mechanism. Anything documented in [ARCHITECTURE.md](ARCHITECTURE.md) under "System Overview" or "Extension Points" requires an RFC.

**Changes the configuration schema in incompatible ways.** Renaming an existing configuration field, removing one, or changing the type of an existing one. Adding a new optional field is not an RFC; making a previously-optional field required is.

**Introduces a new core dependency.** Adding a new package to `pyproject.toml`'s required dependencies. Adding to optional extras (`[dev]`, `[docs]`) does not require an RFC.

**Introduces a new public extension interface.** A fourth abstract interface alongside `DataSource`, `Transform`, and `TaskAdapter` requires an RFC, because every such interface establishes a long-term contract with future plugin authors.

**Changes the project governance.** Modifications to this document, to [CONTRIBUTING.md](../CONTRIBUTING.md), or to the maintainer set.

**Departs from a documented design principle.** Changes that knowingly violate a principle in [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md) require an RFC that explicitly addresses the trade-off and either justifies the violation or proposes amending the principle.

---

## When an RFC is Not Required

The following changes are made through standard pull requests without an RFC:

- Bug fixes that restore documented behavior
- Adding a new plugin (filter, source, adapter) to the existing extension points — these go through the procedure in [CONTRIBUTING.md](../CONTRIBUTING.md#adding-a-new-plugin)
- Performance improvements that preserve external behavior
- Documentation improvements
- Test additions or refactors
- Adding new optional configuration fields with sensible defaults
- Internal refactors that do not change any documented behavior

When in doubt, file an issue describing the proposed change before opening either a PR or an RFC. A maintainer will indicate which path is appropriate.

---

## The RFC Lifecycle

An RFC moves through five states. Each transition is recorded in the RFC document's metadata.

### Draft

A new RFC is initially in **Draft**. The author writes the document, refines it through informal discussion, and prepares it for formal review. RFCs in Draft live as pull requests against `docs/rfcs/` with the status `Draft` in their metadata.

There is no time limit on Draft. An RFC may remain in Draft indefinitely while the author iterates.

### Under Discussion

When the author considers the RFC ready for formal review, they update the status to **Under Discussion** and request review from at least two maintainers. From this point, the RFC is publicly open for community comment.

Discussion happens in two channels: line-level comments on the pull request for specific text, and the RFC's pull request description for high-level threads. All discussion is public and recorded.

The Under Discussion phase has a minimum duration of 14 calendar days from the moment the status changes. This ensures that the broader community has time to read and respond. Maintainers may extend this period if the RFC is complex or controversial.

### Final Comment Period

When discussion has reached a natural pause and a clear direction has emerged, a maintainer may move the RFC to **Final Comment Period (FCP)**. The FCP is a 7-day window during which any final objections must be raised. Substantive new objections during FCP reset the clock to Under Discussion.

Entering FCP signals an intent to decide. It is not a guarantee of acceptance.

### Accepted or Rejected

At the end of FCP, the maintainer team makes a decision:

- **Accepted**: the RFC is merged into `docs/rfcs/` with status `Accepted` and an assigned RFC number. Implementation can begin.
- **Rejected**: the RFC is merged with status `Rejected` and a clear explanation in the document of why. Rejected RFCs remain part of the project history; they are not deleted.

Decisions are made by maintainer consensus when possible. When consensus is not reached, the decision is made by a vote of the maintainer team, with a simple majority required for acceptance.

### Implemented

After an Accepted RFC is fully implemented in code, its status is updated to **Implemented** with a reference to the merge commit or release. RFCs may be partially implemented across multiple PRs; the status updates only when the full RFC is realized.

An Accepted RFC that has not been implemented after 12 months is reviewed by maintainers to decide whether it should be marked Stale, withdrawn, or have its scope reduced.

---

## RFC Document Structure

Every RFC follows the same document structure. The structure is enforced by a CI check that validates the RFC against the schema below.

```text
# RFC <NNNN>: <Title>

| Field | Value |
|---|---|
| Status | Draft / Under Discussion / Final Comment Period / Accepted / Rejected / Implemented |
| Author | <name> <email> |
| Created | YYYY-MM-DD |
| Last updated | YYYY-MM-DD |
| Implementation PR | #<number> (once implemented) |

## Summary

A one-paragraph explanation of the proposal.

## Motivation

Why is this change needed? What problem does it solve? What concrete use cases
or pain points does it address?

## Detailed Design

The technical specification. This is the heart of the document. It must be
detailed enough that a contributor unfamiliar with the proposal could implement
it from this section alone.

## Drawbacks

Why might this proposal be a bad idea? What costs does it impose? What use
cases does it make harder?

## Rationale and Alternatives

Why is this design the best among possible alternatives? What other approaches
were considered, and why were they rejected?

## Prior Art

Discussion of how similar problems are solved in other projects (Rust RFCs,
Python PEPs, related ML pipelines, related astronomy software). Direct
references with links.

## Unresolved Questions

What parts of the design are still open? What questions are explicitly out of
scope but related?

## Future Possibilities

Natural extensions of this proposal that are not part of this RFC. This section
is informative; nothing in it is decided by the RFC.
```

The numbering of RFCs is sequential, assigned at the moment the RFC enters Under Discussion. Drafts have placeholder number `0000`.

---

## Discussion and Decision

The discussion process aims for a written, public, durable record of the reasoning behind every architectural decision in FLUX.

**All discussion is public.** Comments by maintainers carry no more weight than comments by the community. The maintainer role is to facilitate the discussion and ultimately make the decision, not to dominate the conversation.

**Substantive disagreement requires substantive response.** A "I disagree" comment without reasoning may be acknowledged but does not block progress. A reasoned objection — pointing to a use case the proposal breaks, an unaddressed performance concern, or a violation of a stated principle — must be addressed before FCP.

**Authors are encouraged to incorporate feedback.** RFCs evolve during discussion. Authors are expected to update the document in response to substantive comments, not to defend the original draft at all costs. An RFC that did not change between Draft and Accepted has not been properly reviewed.

**The decision rationale is recorded.** When an RFC is Accepted or Rejected, the maintainer making the call adds a note to the RFC document explaining the decision and the key arguments that drove it. This becomes part of the project's institutional memory.

---

## Implementation

After acceptance, the RFC author or any other contributor may implement the proposal. Implementation does not require the original author.

Implementation PRs reference the RFC by number in the commit message:

```
feat: implement bounded async error bus (RFC 0007)
```

Implementation may differ from the RFC in details that emerge during coding. Significant deviations require an amendment to the RFC (see below). Minor deviations — variable naming, internal helper structure — are acceptable without amendment, provided the public interface and behavior match the RFC.

When implementation is complete and merged into `main`, a maintainer updates the RFC's status to **Implemented** and adds a reference to the implementation PR.

---

## Withdrawing or Amending an RFC

**Withdrawal.** An RFC author may withdraw their RFC at any time before acceptance. The status changes to `Withdrawn` and the document is preserved as part of the project's history.

**Amendment of an Accepted RFC.** Once an RFC is Accepted but not yet Implemented, amendments require a new pull request that updates the original document. Substantive amendments (changes to detailed design or motivation) require a new round of review with at least 7 days of discussion. Editorial amendments (typos, broken links, clarifications) can be merged as standard documentation fixes.

**Amendment of an Implemented RFC.** Implemented RFCs are historical records and are not amended. A change to behavior previously specified by an Implemented RFC requires a new RFC that supersedes it. The new RFC explicitly notes which RFC it supersedes, and the superseded RFC's metadata is updated to point to the successor.

This rule preserves the project's ability to trace why current behavior is what it is, by walking the chain of RFCs back to the original justification.

---

## Template

A blank RFC template is maintained at `docs/rfcs/0000-template.md`. To start a new RFC, copy the template:

```bash
cp docs/rfcs/0000-template.md docs/rfcs/0000-my-proposal.md
```

Fill in the metadata header and all sections (using "TBD" or "N/A" where appropriate during Draft), then open a pull request titled `RFC: <Title>` and tagged with the `rfc` label. The numbering placeholder `0000` is updated by a maintainer when the RFC enters Under Discussion.

---

## See also

- [CONTRIBUTING.md](../CONTRIBUTING.md) — standard contribution workflow
- [DESIGN_PHILOSOPHY.md](DESIGN_PHILOSOPHY.md) — principles that RFCs must respect or explicitly amend
- [ARCHITECTURE.md](ARCHITECTURE.md) — the structure that significant RFCs may modify
  - [ROADMAP.md](../ROADMAP.md) — planned changes that may motivate RFCs