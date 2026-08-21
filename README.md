# IntentRail A2A lifecycle extension

**Status:** discussion draft
**Last updated:** 21 August 2026
**Proposed artifact:** `experimental-ext-intentrail`

IntentRail is a proposed A2A extension for tasks that can cause an external
side effect. The aim is to make four things checkable across implementations:

1. what was requested;
2. what was authorized;
3. whether that authorization was already used; and
4. what happened after execution.

This is not a proposal to change A2A task states or to add an identity, policy,
or payment system to A2A. It is about the execution boundary: the point after a
task is received but before an agent is allowed to perform the side effect.

The discussion is tracked in
[a2aproject/A2A#2098](https://github.com/a2aproject/A2A/issues/2098).

## The problem

Authentication tells a receiver who sent a request. Authorization can tell the
receiver that the sender is allowed to do something. Neither one, on its own,
proves that the action executed later is the same action that was approved.

Between approval and execution:

- parameters can change;
- a recipient or resource can be substituted;
- a constraint can expire or be revoked;
- a retry can become a second execution; or
- a completion record can describe an outcome unrelated to the approved action.

Applications can solve this privately with metadata and local state. The
interoperability question is whether two A2A implementations can apply the same
checks and reach the same result.

## Boundary with A2A

A2A continues to define discovery, messages, tasks, task state, and returned
artifacts. IntentRail would add a versioned extension binding around an A2A task:

```text
preflight -> authorization -> execution gate -> outcome -> verification
```

The extension would use existing A2A extension negotiation and namespaced
metadata. It would not add values to A2A core enums or reinterpret an A2A
`taskId`.

## The four linked records

### 1. Preflight proposal

The requester creates a proposal before execution. It contains an objective,
an action manifest, constraints, the requesting party, and a creation time.

Once submitted, the proposal is immutable. A material change creates a new
proposal, optionally linked to the one it replaces. This prevents a later
parameter change from being treated as the action that was originally reviewed.

### 2. Authorization decision

An authorized party accepts, rejects, or counters the proposal. Acceptance
creates an immutable execution contract containing the approved action and its
constraints. The decision records who made it and when.

A deployment may attach signed approvals or a signed decision receipt. The
extension does not prescribe the policy language, credential, or service used
to reach the decision. It requires the execution boundary to verify the
decision that it relies on.

### 3. Execution identity

The accepted contract provides a stable execution identity. The current name
for it is `contractId`.

`contractId` and A2A `taskId` are separate. The contract identifies the
authorized lifecycle; the task identifier keeps its normal A2A meaning. Once a
task is created, the two can be linked in task metadata.

Immediately before dispatch, the receiver checks the stored contract, caller,
constraints, approvals, and current state. It then reserves the contract for
execution atomically. The task handler is called only after that succeeds.

### 4. Outcome record

After execution, the executor records outcome evidence linked to the contract.
The record says what was observed, when it occurred, and which artifacts or
digests support it.

Evidence is not considered true merely because the executor submitted it. A
verification step can complete the contract, mark it failed, or route it into
compensation. The outcome stays linked to the proposal and authorization that
allowed the execution.

## Lifecycle

The main path is:

```text
PROPOSED -> ACCEPTED -> EXECUTING -> COMPLETED -> VERIFIED
```

Other expected paths include:

```text
PROPOSED -> REJECTED
ACCEPTED -> CANCELLED
EXECUTING -> FAILED
COMPLETED -> COMPENSATION_REQUIRED -> COMPENSATED
```

An implementation rejects a transition that is not allowed by the lifecycle.
Successful transitions produce ordered audit records so the current state can
be explained later.

## Initial A2A carrier

The exact wire shape is still open for discussion. A small binding could carry
the contract identifier and the material needed to verify execution
authorization:

```json
{
  "version": "draft",
  "contractId": "ctr_123",
  "authorization": {
    "facts": {
      "environment": "production"
    },
    "approvals": []
  }
}
```

The object would be stored under the extension URI in A2A message metadata.
The same URI would appear in `message.extensions`, and the Agent Card would
advertise support and whether the extension is required.

At the receiving boundary, an implementation would:

1. confirm that the message activated the advertised extension;
2. load the contract from a trusted store;
3. verify that the authenticated caller may execute it;
4. check the contract state, constraints, and approvals;
5. reserve the execution identity; and
6. only then call the A2A task handler.

Proposal identifiers and other trusted linkage are derived from the stored
contract rather than copied from untrusted message metadata.

## Retry and replay

Content integrity and replay protection are different concerns. A digest can
show that a record changed, but it does not show whether the action already ran.

The proposed behavior is therefore stateful:

- an exact transport retry resolves to the existing A2A task or stored result;
- the same idempotency key with different input is rejected;
- an accepted contract can enter execution only once;
- repeated delivery cannot call the task handler again; and
- recovery after a crash must not create a second side effect.

Transport retry resolution belongs in durable A2A task storage. The IntentRail
gate must not redispatch an arbitrary task handler merely because the incoming
binding is byte-identical.

## Canonical records

Canonicalization applies only to IntentRail records that are hashed or signed.
A protocol version must define the exact schema and canonical input for each
such record. A verifier selects that version before checking a digest or
signature.

IntentRail does not try to define a universal content identity for every agent
action or replace identifiers owned by another protocol. The fields that
actually require cross-implementation canonicalization remain an open design
question for the experimental draft.

## Initial conformance cases

The first fixture set should cover at least:

1. a valid accepted contract is dispatched once;
2. missing or invalid authorization is rejected before dispatch;
3. a constraint mismatch is rejected before dispatch;
4. an exact retry returns the existing task without redispatch;
5. an idempotency key reused with different input is rejected;
6. a contract already in execution cannot invoke the task handler again;
7. outcome evidence linked to another contract is rejected;
8. an invalid lifecycle transition leaves stored state unchanged;
9. a failed outcome can enter compensation without losing its original links;
10. a crash between reservation and task creation recovers without a second
    side effect; and
11. a multi-hop authorization cannot silently widen its approved scope.

## Scope

IntentRail would define:

- the extension identifier, version negotiation, and activation rules;
- the minimum lifecycle records and their linkage;
- checks required at the execution boundary;
- retry, replay, and partial-failure behavior;
- outcome linkage and verification states; and
- deterministic errors and conformance cases.

IntentRail would not define:

- agent identity or credential issuance;
- a universal authorization token;
- a policy language or risk-scoring system;
- a payment or settlement protocol;
- a global evidence store; or
- a replacement for A2A transport or task state.

## Related A2A discussions

- [#1769](https://github.com/a2aproject/A2A/issues/1769) covers verifier-side admission and permit, evidence, and receipt artifacts.
- [#1716](https://github.com/a2aproject/A2A/issues/1716) discusses capability enforcement at the skill boundary.
- [#1956](https://github.com/a2aproject/A2A/issues/1956) discusses structured business intent and fulfillment constraints.
- [#1976](https://github.com/a2aproject/A2A/issues/1976) discusses requester acceptance or rejection of task output.
- [#1847](https://github.com/a2aproject/A2A/issues/1847) discusses post-execution receipts linked to upstream authority.
- [#2133](https://github.com/a2aproject/A2A/issues/2133) proposes an exact-message authorization envelope for consequential actions.

These appear related and potentially complementary, but this proposal does not
assume any of them as a dependency. I may have missed another discussion or
misunderstood the intended boundary of one of these issues; pointers and
corrections are welcome.

## Questions for review

1. Is this lifecycle narrow enough to be an A2A extension?
2. Should the A2A binding carry complete authorization material, a stable
   reference, or allow both?
3. Which record fields need canonicalization for independent verification?
4. Which retry, timeout, and crash-recovery cases must be mandatory
   conformance tests?
5. How should a multi-hop agent workflow carry authorization linkage without
   changing A2A task identity?
6. Which outcome fields belong in extension metadata, and which should remain
   ordinary A2A artifacts or external evidence references?

## Request

I am looking for community feedback on the boundary above. If A2A maintainers
agree that it belongs in the extension model, I am also looking for a
maintainer sponsor for an `experimental-ext-intentrail` repository. I am
prepared to maintain the experimental work and its conformance fixtures.

The implementation repository remains private while release and security work
is completed. This public proposal is intended to be reviewable without access
to that code.

## License

Apache-2.0.
