---
title: Credential-Brokered Browser Login
authors:
  - sallyom
created: 2026-07-17
last_updated: 2026-07-17
status: draft
issue:
rfc_pr:
---

# Proposal: Credential-Brokered Browser Login

## Summary

Add a provider-neutral browser credential-broker operation that approves,
fills, submits, and verifies or clears a configured login without exposing
credential values to the model, tool arguments, tool results, transcript, or
approval text. The first backend resolves OpenClaw `SecretRef` values inside an
exclusive browser operation and provides a deliberately limited security
claim. Optional external broker plugins may provide stronger custody; if the
OpenShell plugin is enabled, a future backend could use one-use placeholders
and inspected-egress substitution after OpenShell defines a web credential
grant contract.

## Motivation

A normal browser tool can resolve a password from an OpenClaw `SecretRef` and
fill it into a page without returning the value in its tool result. That keeps
the value out of the model's explicit arguments and response, but it does not
keep the value out of the OpenClaw process, browser DOM, page JavaScript, CDP
clients, extension relays, or later browser-tool calls. Describing that flow as
agentic credential protection would overstate its security boundary.

There is still useful value in a narrower built-in operation. OpenClaw can keep
the secret out of model-visible surfaces, require human approval for an
independently observed destination, exclude other browser controllers, submit
without yielding back to the agent, and clear filled values before releasing
the browser on failure. That is materially safer than exposing a generic
fill-only action, even though OpenClaw and the page still receive the value.

Some credential custodians can provide a stronger boundary. Instead of
returning a secret, a broker can return opaque, one-use placeholders and
release the real value only into an approved request after checking the
destination and request shape. OpenShell's inspected egress is a candidate for
that design, but its current reusable provider placeholders are not a
per-login authorization grant. This RFC defines the OpenClaw consumer contract
and requires a separate upstream OpenShell contract before such a backend can
claim that the credential stays out of OpenClaw and the browser DOM.

[1Password for Claude's documented security model](https://support.1password.com/1password-claude-security/)
is a useful reference outcome, not a dependency or an available integration
contract. Any future password-manager backend must use the custodian's
supported application and partner interfaces rather than inferring a private
protocol.

## Goals

- Give the model a login-alias operation without accepting usernames,
  passwords, one-time codes, or secret references in model-supplied arguments.
- Derive the destination and form state from the controlled browser rather
  than trusting model- or page-supplied labels.
- Require a critical human approval that identifies the configured login alias
  and observed destination without displaying secret values.
- Make approve, resolve or grant, fill, submit, and verify-or-clear one bounded
  browser operation.
- Exclude other OpenClaw browser tools, snapshots, evaluations, extension
  relays, and event observers from the browser profile during that operation.
- Keep credential values out of prompts, tool arguments and results,
  transcripts, approvals, and logs for every backend.
- State separately what the built-in SecretRef backend and any external broker
  backend protect.
- Let an enabled broker plugin implement stronger custody without embedding its
  provider ids, placeholder format, or release policy in OpenClaw core.
- Fail closed when the browser profile cannot be made exclusive, the form or
  destination is ambiguous, submission cannot be bounded, or cleanup cannot be
  guaranteed.

## Non-Goals

- Defining session workers, worker providers, sandbox-per-session lifecycle, or
  local model inference.
- Claiming that the built-in SecretRef backend keeps credentials out of
  OpenClaw, the browser DOM, or destination page JavaScript.
- A general-purpose secret-reading browser tool or fill-only credential action.
- Copying external broker credentials into OpenClaw's secret store.
- Treating reusable environment placeholders as per-login authorization.
- Defining OpenShell's web credential grant API; OpenShell must own that
  companion contract.
- Claiming support for a password manager without its supported integration
  contract.
- Supporting passkeys, WebAuthn, PAKE, native-app handoff, uninspectable SSO,
  or client-side password transformations through proxy substitution.
- Protecting against a compromised operator host, browser binary, credential
  custodian, broker control plane, or approved destination.

## Proposal

### Provider-neutral browser broker capability

The browser plugin owns one sensitive browser operation. A configured login
alias identifies a broker backend and backend-owned credential reference. The
model may select the alias and identify visible fields or an intended submit
action, but it never supplies credential values, `SecretRef` objects, an
authoritative destination, or broker grant parameters.

Before approval, the browser plugin independently reads the current top-level
HTTPS origin and the candidate form's effective submission shape. It validates
the configured origin allowlist and presents a critical approval containing the
login alias and observed destination. A backend may add redacted policy
metadata, but it cannot replace the browser's observed origin with a
model-supplied claim.

After approval, the browser plugin acquires an exclusive lease for the entire
browser profile. While held, every other OpenClaw-controlled browser action,
snapshot, evaluation, extension relay, and event observer for that profile is
rejected or queued. The broker operation does not yield control to the agent
between credential placement and submission.

The operation ends only after one of these outcomes:

- submission succeeds and navigation replaces the credential form;
- submission succeeds and every value or placeholder placed by the broker is
  cleared before agent access resumes; or
- failure, timeout, cancellation, or ambiguity causes cleanup before the
  profile lease is released.

If exclusivity cannot be established or cleanup cannot be guaranteed, the
operation rejects before resolving a SecretRef or requesting an external
grant. The tool result contains only a status and redacted metadata.

### Built-in SecretRef backend

The first backend is implemented within OpenClaw's browser plugin and uses
already configured `SecretRef` providers. After approval and profile exclusion,
it resolves the configured username, password, or one-time code internally,
fills the selected fields, submits, and waits for the bounded outcome.

This backend can claim:

> OpenClaw does not intentionally serialize credential values into model
> prompts, model-supplied tool arguments, tool results, transcripts, approval
> text, or logs.

It cannot claim that the values never reach OpenClaw or later model context.
The browser plugin resolves them, and the browser DOM and destination page
JavaScript can observe them during the fill-and-submit window. Page code may
retain or render a value after submission, and another same-user process with
direct browser or process access is outside this boundary.

The backend does not expose a fill-only mode. If it cannot identify a submit
action and bounded success or failure condition, it rejects the request before
secret resolution.

### External credential broker backend

The browser plugin may delegate the same operation to a plugin-provided broker
without asking that broker to return credential values. An external backend
returns only opaque placement material and redacted policy metadata, then owns
credential custody, authorization, release, expiry, revocation, and secret-safe
audit state.

Backend selection is explicit. Failure of an external backend never silently
falls back to the built-in SecretRef backend because that would change the
credential exposure claim after approval.

The exact delivery mechanism is backend-owned. A supported password manager
might use a trusted browser extension or native fill surface. A proxy broker
might return opaque placeholders that are replaced after the browser emits one
approved request. Core and the browser plugin consume only the generic
sensitive-operation outcome; they do not understand vendor placeholder formats
or credential identifiers.

### Optional OpenShell broker backend

If the OpenShell plugin is installed and enabled, a future browser broker
backend may request an OpenShell web credential grant. This integration is
independent of OpenShell worker-provider and model-inference functionality. It
requires OpenShell to prove that the relevant browser profile's network egress
crosses the inspected proxy; a remote CDP connection alone is transport, not
that security boundary.

The intended flow is:

```text
OpenClaw browser plugin
  │  observed origin, form, and approved login alias
  ▼
enabled OpenShell broker plugin
  │  request one scoped web credential grant
  ▼
OpenShell Gateway
  │  return opaque username/password placeholders, never values
  ▼
exclusive browser-profile operation
  │  fill placeholders and submit without yielding
  ▼
OpenShell inspected egress proxy
  │  verify bindings, substitute once, consume grant
  ▼
approved HTTPS login endpoint
```

A reusable provider environment placeholder is not sufficient. From
OpenClaw's perspective, an OpenShell web credential grant must be:

- authorized by a human for one named credential item and displayed
  destination;
- bound to one OpenShell Gateway, browser execution boundary, OpenClaw session,
  browser profile, and browser tool call;
- bound to the exact HTTPS scheme, host, port, request method, path, supported
  content type, field names, and credential placement;
- short-lived, exactly once consumable, and invalid after success, failure,
  cancellation, timeout, policy reload, execution-boundary deletion, or
  session end;
- represented by opaque, non-secret, unguessable placeholders that reveal no
  provider environment key or credential value;
- resolved only for whole supported field values after policy authorization;
  and
- fail-closed on a missing binding, changed destination, unsupported body,
  unresolved placeholder, ambiguous form, concurrent use, or replay.

OpenShell owns grant creation, credential lookup, substitution, consumption,
expiry, and secret-safe audit metadata. OpenClaw owns the user request,
independent browser observation, profile exclusion, browser orchestration, and
redacted result. Neither side logs credential values or a reusable
representation of them.

For a compatible login submitted through this backend, the intended claim is:

> The username, password, and one-time code are not exposed to the model,
> OpenClaw, its tool protocol, or the browser DOM. OpenShell releases them only
> into one approved request to the bound destination.

This claim includes the trusted OpenShell credential store, supervisor, and
inspected proxy in the trusted computing base, and it ends at the approved
destination. It is unavailable until OpenShell defines and implements the
required grant and enforced-egress contracts.

### Proxy-substitution compatibility

An inspected-proxy backend can support only requests whose approved credential
fields carry placeholders unchanged. Initial candidates are ordinary native
form submissions and fetch/XHR requests using approved UTF-8 JSON,
form-URL-encoded, or text fields.

The backend must reject sites that hash or encrypt the password in page
JavaScript, require the real value for client-side validation, use multipart or
binary credential submission, or use passkeys, WebAuthn, PAKE, native-app
handoff, or an uninspectable SSO flow. It never falls back to filling a real
secret into the DOM. Compatibility is documented per backend and login shape.

### Ownership and delivery sequence

OpenClaw owns:

- the provider-neutral browser broker capability;
- login-alias selection and independently observed destination state;
- critical approval presentation;
- exclusive browser-profile access and atomic submit-or-clear behavior;
- the built-in SecretRef backend and its narrower security claim; and
- backend-specific user documentation and redacted status reporting.

Each external broker owns its custody and release contract. For OpenShell, that
includes the grant API, caller and browser identity model, credential storage or
external-vault binding, placeholder rules, expiry, one-use consumption,
inspected-proxy substitution, enforced egress, and secret-safe audit events.

Delivery remains incremental:

1. Land the generic browser broker seam, exclusive profile operation, and
   built-in SecretRef backend with approve, submit, and verify-or-clear proof.
2. Open the upstream OpenShell feature issue and define the web credential grant
   and enforced-egress contracts with OpenShell maintainers.
3. Add the optional OpenShell broker backend only after those contracts exist,
   with a controlled-site end-to-end proof.
4. Document compatible login shapes and expand them only when the selected
   backend's invariant and fail-closed behavior remain proven.
5. Add other custodians only through their supported integration and
   application-identity contracts.

## Rationale

Keeping the browser broker independent avoids coupling a generally useful
OpenClaw safety feature to any sandbox, worker, inference provider, or
credential custodian. The built-in backend provides immediate value and a clear
migration path while preserving an honest, narrower claim.

The main benefits are:

- the model never handles raw credentials or secret references;
- approval is tied to independently observed browser state;
- exclusive submit-or-clear orchestration reduces accidental disclosure after
  fill;
- backend-specific security claims remain explicit; and
- stronger external custody can be added through plugins without placing
  vendor policy in core.

The tradeoffs are:

- the built-in backend still exposes credentials to OpenClaw, the browser DOM,
  and page JavaScript;
- whole-profile exclusion may delay or reject concurrent browser work;
- reliable success detection and cleanup vary across sites;
- proxy substitution has deliberately narrow compatibility;
- an external backend adds another trusted control plane and must prove browser
  placement or enforced egress, not only API connectivity; and
- the strongest OpenShell claim depends on an upstream grant contract that
  does not exist yet.

A generic secret-fill tool would be simpler but would let the agent pause,
inspect, or navigate while a real value remains in the page. Putting
OpenShell-specific grants directly in core would make one optional custodian's
policy part of every OpenClaw installation. Silent fallback between backends
would be worse: it would preserve task progress by invalidating the approved
security boundary.

## Unresolved questions

- What browser profile exclusion primitive covers browser tools, extension
  relays, event observers, and other CDP clients without deadlocking cleanup?
- Which observed form and navigation signals are sufficient to approve and
  recognize successful submission without exposing page-rendered secrets?
- Which login shapes should the built-in backend support initially?
- How should multi-page username, password, and one-time-code flows divide
  approval, exclusivity, and cleanup?
- What versioned OpenShell API should create, revoke, inspect, and consume a web
  credential grant without returning credential values to the plugin?
- How can an enabled OpenShell broker prove that the selected browser profile
  cannot bypass inspected egress?
- Should the trusted approval UI be owned by OpenClaw, an external broker, or a
  coordinated two-party flow?
- What portable broker result is sufficient for browser success reporting and
  audit without flattening backend-specific guarantees?
