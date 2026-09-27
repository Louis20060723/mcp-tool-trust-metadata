# Trusted Metadata for MCP Tools

A working paper on why an agent's tool supply chain needs **provenance** before it needs
permissions — and how to provide it without inventing a new format or trusting a registry.

**Read the paper:** [`paper.md`](paper.md)

## The short version

Agents discover and call tools on remote MCP servers. Users cannot see who published a
tool, what it declares it does, or what authority it exercises. Today's checks happen at
call time; by then the decision to connect has already been made.

The US NSA's May 2026 guidance (PP-26-1834) names this gap and calls for origin
verification, authorization checks and a registry carrying provenance, version and known
issues. This paper takes the provenance half seriously:

1. **Reuse SPDX 3 / CycloneDX** instead of inventing another format.
2. **Verify offline** — a client holding a signed statement must be able to check it
   without a round trip to any authority.
3. **Declared capabilities must be checkable fields**, not prose.
4. **Fail closed** — "unverified" is a first-class state.
5. **A registry is for discovery, not for truth.**

## Status

Working paper, September 2026. Related implementation work in progress; see the paper's
status section for what has landed upstream and what is still open.

## Licence and reuse

Text may be quoted and reused with attribution to this repository.

---

*The author is the maintainer of github.com/Louis20060723. This paper was drafted with AI
assistance; the technical positions, editorial decisions and any errors are the author's.*
