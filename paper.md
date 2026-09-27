# Trusted Metadata for MCP Tools

## Why an agent's tool supply chain needs provenance before it needs permissions

*Working paper — September 2026*

---

## Summary

Agents now discover and call tools on remote MCP servers without any reliable way to
learn who published a tool, what it claims to do, or what authority it will exercise.
Current practice checks authority *at call time* (tokens, scopes, allow-lists). That is
necessary but not sufficient: at call time the user has already decided to connect, and
every decision downstream inherits that decision.

This paper argues that the missing layer is **provenance** — verifiable, machine-readable
facts about a tool that a client can evaluate *before* the first call, without trusting
the registry that served them.

Two design commitments follow. First, **reuse existing supply-chain standards** (SPDX 3,
CycloneDX) rather than inventing a new format; MCP tooling should look like the rest of a
software supply chain, because that is where the tooling, scanners and reviewer habits
already exist. Second, **verification must work offline**: a client that holds a signed
statement about a tool must be able to check it without a round trip to any authority.

---

## 1. The problem

An agent asked to "clean up these files" resolves a tool by name, connects, and calls it.
The user sees an outcome, not a supplier. Three questions are unanswerable with today's
tooling:

1. **Who published this tool?** A server URL and a self-declared title are not identity.
2. **What does it declare it will do?** Descriptions are prose written for a language
   model, not a contract that can be checked.
3. **What authority does it exercise?** Filesystem, network egress, credentials, spend —
   none of this is visible before the call.

The consequence is not hypothetical. Dynamic tool discovery means the set of tools a
client may call is not fixed at install time; it can change per session and per request.
Any static review of "the tools this agent has" is therefore obsolete the moment it is
written.

## 2. Why now: the guidance already asks for this

In **May 2026** the US National Security Agency published *MCP Security Guidance*
(**PP-26-1834**, 15 pages). Its framing is unusually blunt for a government document:

- "MCP has become the de facto standard" for connecting models to tools;
- "its rapid adoption has outpaced the development of appropriate security safeguards";
- dynamic tool discovery must be treated with care, "unless it can be coupled with
  **origin verification** or **authorization checks**".

The same document calls for a **registry of MCP tools carrying provenance, version and
known issues**, and it sets a compliance expectation for federal contractors in 2026.

Read carefully, this splits the problem in two:

| Question | Layer | Where it is answered |
|---|---|---|
| Is this tool *allowed* to do that to me? | Authorization | runtime (tokens, scopes, policy) |
| Is this tool *what it says it is*? | Provenance | before runtime (metadata, signatures) |

Most current effort is on the runtime half. This paper is about the other half — the one
that has to be settled before a runtime check can mean anything.

## 3. Design principles

**P1 — Do not invent a format.** SPDX 3 and CycloneDX already model suppliers, versions,
components, relationships and attestations. A parallel MCP-specific format would fragment
review, tooling and scanner support. Profile, do not fork.

**P2 — Offline verification is the requirement.** A client must be able to validate a
tool's metadata with only local material (the statement, its signature, the public key or
transparency proof). If verification requires calling the issuer, then the issuer is the
trust root and the metadata adds nothing.

**P3 — Declared capabilities must be checkable.** "Declares read-only access to a given
directory" is useful only if it is a field, not a sentence — and only if a client can
compare the declaration against observed behavior afterwards.

**P4 — Fail closed, degrade loudly.** Missing or invalid provenance should be a
first-class state (unknown / unverified / verified), never silently treated as trusted.
Absence of metadata is information.

**P5 — Absence of a registry must not be fatal.** Registries are convenient discovery
points, not authorities. If a tool is not listed anywhere, that should reduce confidence,
not block the model of verification entirely.

## 4. A metadata profile for MCP tools

The profile reuses SPDX 3 / CycloneDX structures and adds only what is MCP-specific:

**Identity and origin**
- publisher identity (organisation or individual) with a verifiable key or OIDC issuer
- source repository and the exact revision the published tool was built from
- build attestation: who built it, from what commit, with what toolchain

**Tool-level facts**
- tool name and a stable identifier, versioned
- declared capabilities: filesystem scopes, network egress destinations, credential and
  spend access, subprocess execution — expressed as enumerable fields
- declared side effects (idempotent, destructive, externally visible)
- security contact and a vulnerability disclosure path

**Supply-chain linkage**
- SBOM reference for the server implementation
- references to upstream components with versions
- known-issues / advisory links, so "version + advisory" is answerable offline

**Signature**
- detached signature over the statement, with a documented canonicalization so that
  third parties can re-implement verification

## 5. Verification model

Verification has three independent steps, each of which can fail on its own:

1. **Signature validity** — the statement is signed by the key it claims, and the
   statement bytes are the ones signed.
2. **Attribution** — the key is bound to a publisher identity in a way the client can
   check offline (a certificate chain, or a transparency-log inclusion proof).
3. **Freshness and revocation** — the client is not accepting a stale or withdrawn
   statement; this needs a bounded-age rule rather than an online check.

A client that can complete these three steps holds something it did not have before: the
ability to say "this tool comes from this publisher, from this source revision, and
declares these capabilities" — and to record that statement alongside what the agent
actually did, so the two can be compared later.

## 6. What a registry is for, and what it is not

A registry should answer *discovery* and *distribution* questions: where do I find the
statement, which versions exist, what advisories are attached. It should not be the thing
that makes metadata true. If a registry can silently substitute metadata, the signature
model is decorative.

The practical form is therefore: statements published by publishers in their own
repositories and release artifacts, aggregated (optionally federated) into catalogues that
anyone can mirror, with the verification path remaining local.

## 7. Adoption path

An honest adoption path for something like this has four rungs, and the first two carry
most of the value:

1. **Emit**: producers publish a signed statement alongside the tool, generated in CI from
   the same revision that produced the artifact.
2. **Check in CI**: a reusable action that fails a build when declared capabilities do not
   match the implementation's observed behavior in tests.
3. **Surface**: clients display provenance and capability declarations at connect time,
   including the "unknown" state.
4. **Aggregate**: catalogues, advisory feeds, mirroring.

Steps 1 and 2 are achievable by a small number of projects and are independently useful.
Step 3 changes user-visible behavior, and step 4 is infrastructure.

## 8. Status and evidence

This is a working paper tied to implementation work in progress, not a finished standard:

- The ideas above align with implementation and funding proposals currently in progress,
  covering runtime authorization and pre-runtime metadata respectively.
- Related work has already landed upstream in Bernstein (Apache-2.0, an agent governance and
  orchestration framework): twenty-two merged pull requests by the author, covering
  supply-chain metadata, replay lineage and quality gates.
- Two further changes are in review: an AI-disclosure assertion in the C2PA projection
  (#6278) and a CycloneDX 1.7 BOM carrying the ML-BOM model card (#6279).

The pieces that remain open are the ones this paper deliberately does not hand-wave:
capability vocabularies that are expressive enough to be useful yet small enough to be
checked, revocation that does not require an online lookup, and the incentives that make a
publisher want to sign anything at all.

## 9. What to do on Monday

If you run an agent, or publish tools for one:

1. Record, per call, the tool identity, version and the statement you verified — even if
   the statement is "none". Retrofitting this later is expensive; the data is cheap now.
2. Ask your tool suppliers to publish a signed statement from CI. That single artefact
   unlocks everything downstream.
3. Treat "unverified" as a first-class state in your client, and show it to the user.

---

*The author is the maintainer of github.com/Louis20060723. This paper was drafted with AI
assistance; the technical positions, editorial decisions and any errors are the author's.*
