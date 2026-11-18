# Proposal: Flagverse — reproducible testing of feature flag interactions

Status: proposed for discussion. This document describes an unimplemented integration; it does not announce a supported
Flagsmith feature or request an implementation merge.

## Problem

A user experiences a combination of feature flags. Testing each feature independently can miss failures that appear only
when features interact. For example, express checkout and coupon discounts might each work independently, while enabling
both submits an undiscounted total.

Four Boolean flags produce 16 possible configurations; ten produce 1,024. Teams need a bounded way to investigate these
interactions, reproduce a failure, and verify a mitigation without changing their live environment.

## Proposed outcome

Flagverse would be a companion developer tool that runs the same browser journey against selected Flagsmith
configurations. Its interface would display each configuration as a small replay card, with the evaluated flag values,
the assertion result, and a link to the execution evidence. A failed card would lead to a reduced reproduction and a
reviewable mitigation.

The proposed workflow is:

1. Select a sandbox project, up to four Boolean flags, a baseline configuration, and one checkout journey.
2. Review the agent's test plan and approve creation of the sandbox test identities and overrides.
3. Run the journey against the selected configurations, using Flagsmith to evaluate the flags consumed by the
   application.
4. Investigate a failure by testing smaller changes from the baseline under the same application revision and fixtures.
5. Review a proposed Flagsmith configuration change, approve its application in the sandbox, and rerun the relevant
   checks.

This proposal seeks feedback on the integration boundary and evidence model. An initial prototype could live in a
separate repository and use existing public interfaces, without adding a runtime dependency to Flagsmith's API or
dashboard.

## Initial scope

The first prototype would support one instrumented demo application, one explicit checkout journey, four Boolean flags,
and at most 16 configurations per investigation. Its fixture would contain an intentionally introduced interaction
between `express_checkout` and `coupon_discounts`; `regional_pricing` and `new_cart_layout` would provide unrelated
combinations. The fixture and seeded bug would be documented as demonstration data.

The checkout would use a payment stub and assert the persisted order total against an independently calculated expected
total. A success banner alone would not establish a passing result. Each run would reset the cart, orders, browser
session, and test identity so another run could not change its outcome.

Out of scope for this prototype are arbitrary applications, production writes, automatic source-code repair,
multivariate flags, and exhaustive testing of large flag sets. Passing these checks would establish only the tested
behaviour under the recorded conditions. It would not establish release safety for all users or predict conversion
rates.

## Flagsmith integration and isolation

Flagsmith would remain responsible for flag storage, targeting, and evaluation. The integration would use the documented
[Management API](../docs/integrating-with-flagsmith/flagsmith-api-overview/management-api/index.md) for authorised
configuration operations and an SDK for the application's evaluated flag values. It would not implement an alternative
evaluation engine.

The prototype would use a dedicated sandbox project containing no production identities or real customer data. This is
deliberate: segment definitions are shared at project scope, so a separate environment alone is not sufficient isolation
for all proposed configuration writes. The tool would act only on its explicitly configured sandbox project and
application URL. Credentials would stay in the local runner, outside browser bundles, replay artefacts, and source
control.

Each configuration would have a unique test identity with identity overrides for the selected flags. Before running the
journey, the application would record the effective flag values it actually consumed. An unexpected configuration, SDK
fallback, or evaluation error would make the run invalid rather than passing. The prototype would initially require
simple Boolean flags without prerequisite or segment rules in the exploration fixture.

[Identity overrides take precedence over segment overrides](../docs/flagsmith-concepts/segments/index.md), and they also
take precedence over [flag prerequisites](../docs/managing-flags/flag-dependencies.md). Forced identity configurations
therefore exercise application behaviour but do not validate a production targeting policy. The UI and report would make
that distinction explicit.

Mitigation verification would use fresh identities without forced overrides for the affected flags, so the proposed
segment rule could actually take effect. For example, after approval, the tool could add a sandbox rule disabling
express checkout for the application's `has_coupon` trait. It would then verify both coupon and non-coupon journeys
through normal SDK evaluation, including that non-coupon users still receive express checkout. The application would
need to refresh the trait and evaluate the flag when the coupon changes; the rule would not discover cart state by
itself.

## Execution and evidence

The runner would use a fixed browser journey and explicit assertions. An AI agent could select candidate configurations,
explain evidence, and propose a mitigation. An agent's explanation would not determine pass or fail.

| Component         | Responsibility                                                                                  |
| ----------------- | ----------------------------------------------------------------------------------------------- |
| Planner           | Produce a bounded configuration plan and explain its purpose for human review.                  |
| Flagsmith adapter | Apply approved sandbox changes and record configuration before and after each change.           |
| Browser runner    | Execute the fixed journey with isolated fixtures and capture assertions and traces.             |
| Report view       | Present replay cards, configuration differences, reduced reproductions, and approval decisions. |

Each run record would include the application revision, journey revision, fixture seed, sandbox environment, test
identity, requested configuration, effective flag values, expected and actual assertion values, and trace location. It
would also record the relevant configuration before and after execution; a change during a run would invalidate the
result. SDK caching would need to be refreshed before the effective values were checked.

Results would distinguish passed, failed, invalid, and not run. A timeout or unavailable backend would not be silently
treated as an application interaction failure. Replay evidence would use synthetic data and omit credentials and session
tokens; exporting a trace would require checking its network and storage contents for secrets.

To reduce a failure, the runner would evaluate subsets of the flag changes relative to the recorded baseline. For this
small fixture it could enumerate all subsets and establish the smallest reproducing change within the tested set.
Repeated runs would check that the same assertion fails consistently. If it did not, the report would mark the result as
unstable and omit a causal claim. Larger, budget-limited searches would have to report a reduced example rather than
claim a globally minimal one.

## Approval and change lifecycle

Before each configuration mutation, the tool would show the sandbox project and environment, affected flags or segment,
current values, proposed values, and the evidence motivating the change. Approval would apply to that concrete change
only. The adapter would re-read the relevant state before applying it and require a new review if it had changed.

An approved mitigation would be marked verified only after checking the effective values and rerunning the affected and
unaffected fixture journeys. A successful API response alone would not complete verification. Failed verification would
be reported with the applied configuration and a proposed restoration of the captured prior state. Restoring that state
would also require approval and a drift check.

The runner would track all identities and configuration objects it creates. Cleanup would be limited to those tracked
resources, with a reviewable plan; it would never delete unrelated project resources. API calls would respect the
deployment's rate limits, and the run budget would cap configurations, retries, and concurrent browser sessions.

## Relationship to Experimentation

Synthetic browser checks answer whether a specified journey behaves as expected. They do not answer which variant real
users prefer. Test identities and events would stay in the sandbox and would not be added to a real experiment's data.

A later extension could prepare a native Flagsmith experiment for technically verified variants and request approval
before starting it. Flagsmith would continue to own allocation and statistical analysis. That extension is outside the
initial prototype, and its results would remain separate from the QA report.

## Acceptance criteria for a future prototype

- All 16 fixture configurations use flags evaluated by Flagsmith and display the values consumed by the application.
- The seeded checkout interaction fails the order-total assertion and produces a trace that reproduces it with the same
  application revision and fixtures.
- Reduction identifies the two relevant flags in the fixture; testing each change alone passes the same assertion.
- No configuration mutation occurs before approval, and approval cannot be reused after the proposed change or baseline
  state changes.
- Verification of the approved segment mitigation uses normal targeting, fixes the coupon journey, and preserves express
  checkout for the non-coupon journey.
- Invalid or unstable runs are reported explicitly. Production resources and real experiment datasets remain unchanged.

These are proposed checks for an implementation, not completed test results for this document.

## Alternatives and related work

A manually maintained browser-test matrix can cover known combinations with less tooling. Pairwise generators can reduce
the number of cases for larger flag sets, but do not establish coverage of every higher-order interaction. Flagverse
would build on those testing techniques; its proposed contribution is the connected investigation, reproduction,
approval, and verification workflow.

[Issue #9006](https://github.com/Flagsmith/flagsmith/issues/9006) proposes an integration for independently verified
feature-flag experiments. This proposal overlaps in its separation of technical verification and experiment results,
while focusing specifically on interactions between flags in an existing application and reducing a failing
configuration. It does not assume acceptance of, or an implementation dependency on, that proposal.

## Questions for maintainers

1. Would a standalone integration be the preferred home for this prototype, and should this proposal move to an issue or
   discussion instead of remaining in the repository?
2. Are the proposed sandbox identity overrides an appropriate boundary for the initial fixture, with normal-targeting
   checks kept separate?
3. Which existing configuration-change or approval interfaces should an eventual integration reuse before considering
   any additional Flagsmith API surface?

This document is under `docs/proposals/`, outside the Docusaurus content directory (`docs/docs/`), so it is not
presented as published product documentation.
