# Security and Access-Control Plan

This document outlines intended security boundaries; the repository currently
has no authentication, authorization, encryption, or secret-management
implementation.

## Secret handling

- Never commit credentials, API tokens, private keys, certificates, or
  production connection strings.
- Obtain secrets at runtime from an approved secret manager or protected
  environment injection. Keep only non-sensitive variable names and
  placeholders in example configuration.
- Restrict secret access to the process role that needs it, rotate credentials,
  and ensure logs and error reports redact secret values.
- Do not create a repository directory for live keys. If a deployment requires
  mounted secret files, mount them outside the repository with restrictive
  ownership and permissions.

## Access boundaries

The future policy templates belong under `config/security/`; enforcement
interfaces belong under `src/platform/security/`. Define separate permissions
for source consumption, output delivery, shared state and message-bus access,
backup access, and control-plane administration. Follow least privilege and
record security administrative actions without logging secret material.

## API identity and authorization lifecycle

The control API contract uses bearer authentication as a placeholder; select an
identity provider and workload authentication mechanism before implementation.
Prefer short-lived workload identity over static node credentials. Bind each
identity to an approved cluster, node generation, role, and allowed API
operations. A client-provided node ID, role, or zone is a claim to validate,
not proof of identity.

Apply authorization at every operation and resource boundary:

| Principal | Allowed actions |
| --- | --- |
| Worker node | Register/heartbeat as its bound identity, read and acknowledge only its current assignments, request drain for itself |
| Control-plane service | Read membership, issue assignment epochs, revoke/fence owners, coordinate drain |
| Operator | Approved administrative actions through an audited identity; no shared worker credentials |
| Health probe | Read only liveness/readiness status; no topology or secret details |

Credential lifecycle:

1. Provision identity and least-privilege policy through the platform's
   identity provider; do not mint identities from unchecked registration
   payloads.
2. Obtain short-lived credentials at workload start and renew before expiry
   using the identity provider.
3. During rotation, accept old and new credentials only for a bounded overlap;
   confirm the new credential works, then revoke the old one.
4. On node drain, decommission, or suspected compromise, revoke identity,
   invalidate sessions/tokens, and fence all assignment epochs associated with
   the node generation.
5. Audit issuance, renewal, rotation, revocation, authorization failures, and
   administrative assignment changes without logging tokens.

Require TLS for API traffic, validate audience/issuer/expiry and cluster scope,
and protect against replay with request freshness or idempotency semantics.
Rate-limit registration and heartbeat calls. Return minimal errors that do not
reveal whether a protected node or assignment exists to unauthorized callers.

## Data protection

Restrict access to cluster APIs, message topics, shared state, object storage,
logs, and telemetry. Define encryption in transit for node-to-node and external
connections, and encryption at rest for shared state and backups. Redact or
minimize personal and credential-bearing payload data in logs, quarantine, and
diagnostic output.

Before deployment, define a threat model, identity source, credential rotation
procedure, audit requirements, and incident response process. These directories
alone do not enforce any security control.
