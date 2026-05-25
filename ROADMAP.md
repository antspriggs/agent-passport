# Roadmap

This file is a list of **candidate** next steps, not commitments. Each entry
explains the value, the rough effort, and what's currently blocking it.
Discussion happens in GitHub issues; this file is the index.

Ordering is by my (the maintainer's) current sense of priority, not by
size. Priorities will shift as feedback arrives.

## Strategic

### Engage with the NIST AI Agent Standards Initiative

**Value:** the project's stated mission is to contribute to the [NIST AI
Agent Standards Initiative](https://www.nist.gov/artificial-intelligence/ai-agent-standards-initiative)
and the [NCCOE Software and AI Agent Identity and
Authorization](https://www.nccoe.nist.gov/projects/software-and-ai-agent-identity-and-authorization)
project. We now have a real, runnable, NIST-grounded implementation —
this is the moment to engage.

**Concrete actions, ordered by cost:**

1. **Write a 1-page brief** mapping every claim/check in this library to
   the specific NIST SP 800-63-3 / RFC 8693 / OIDC clauses it implements.
   Material exists in [CLAUDE.md](https://github.com/antspriggs/nist-agent-passport/blob/main/CLAUDE.md);
   needs distillation for a non-implementer audience.
2. **Submit a comment** to any open NIST RFI / RFC about agent identity
   pointing at this implementation as a concrete reference.
3. **Propose the namespaced claim schema** (`https://agent-passport.org/claims/`)
   for inclusion in whatever registry NIST or OpenID Foundation maintains
   for agent-identity claims.
4. **Get a NIST or NCCOE reviewer** to walk the codebase against their
   threat model and file findings.

**Status:** unblocked; needs maintainer time.

### Second-CSP validation

**Value:** the project is generic OIDC + PKCE, but only ID.me has been
exercised end-to-end against the live wire. A second CSP would either
confirm or surface portability bugs.

**Candidates** (in order of expected ease):
- **Login.gov sandbox** — the federal-government NIST-aligned CSP. Most
  natural fit; expected to emit canonical `…/ial/N` URIs that our
  default mapping already handles.
- **Auth0 free tier** — universally available; tests the `0` IAL path
  more thoroughly (Auth0 typically doesn't emit `acr` for username/
  password auth).
- **Keycloak self-hosted** — tests against an open-source CSP we can
  configure ourselves; valuable for the hermetic-integration story.

**Status:** unblocked; same shape as the ID.me integration we just did.

## Library features

### Token revocation (RFC 7662 introspection)

**Value:** today the only defense against a compromised token is its TTL
(default 15 min for root, 5 min for child). For high-assurance
deployments that need to invalidate a token immediately on detected
compromise, we need an introspection endpoint per [RFC
7662](https://datatracker.ietf.org/doc/html/rfc7662) and a revocation
list the verifier can consult.

**Design sketch:**
- Issuer maintains a revocation list keyed by `jti` (could be a Bloom
  filter for size, or a small DB).
- Verifier consults the revocation list on each verify, with a short
  cache TTL so revocations propagate in seconds not minutes.
- RFC 7662 `POST /introspect` endpoint on the issuer; takes a token,
  returns `active: true/false` + claim metadata.

**Status:** designed; significant implementation effort; flagged in
[CHANGELOG.md](https://github.com/antspriggs/nist-agent-passport/blob/main/CHANGELOG.md)
Known Limitations since v0.0.1.

### JWKS hosting and key rotation

**Value:** v0 ships a single-issuer single-key model. Real deployments
need to rotate signing keys without invalidating in-flight tokens (overlap
period with old + new keys both publishable), and to publish the JWKS
over HTTPS for verifiers to fetch on `kid` miss.

**Design sketch:**
- Issuer supports an ordered key list (newest first); signs with newest,
  verifies against any of them.
- JWKS endpoint at `<issuer>/.well-known/jwks.json` serves all currently-
  valid public keys.
- Rotation = add new key to the list (signs going forward), retire old
  key after max-token-TTL has elapsed.
- Verifier's `KeyStore` Protocol grows a JWKS-fetching implementation
  (today only the in-memory `InMemoryKeyStore` exists).

**Status:** designed; medium effort.

### Full browser-OAuth integration test

**Value:** the CLI's `login` (OAuth code + PKCE) is exercised via the
paste-in `--id-token` path in CI. The full browser dance is not — we
caught the CLI-default bug only because of the live ID.me integration.
Adding `/authorize` and `/token` endpoints to the mock OIDC provider
would let us test the full dance hermetically in CI on every PR.

**Effort:** ~half a day. The mock provider already implements discovery
+ JWKS; adding the OAuth endpoints is well-understood mechanical work.

**Status:** unblocked; small.

### CLI PII redaction in `inspect`

**Value:** the ID.me integration revealed that `nist-agent-passport
inspect` happily prints legal names, emails, residential addresses, etc.
when the ID token carries them. For a tool people will run in shared
terminals (or paste into shared chats), default-redact-with-opt-in is
the safer posture.

**Design sketch:**
- Default: redact OIDC's standard PII claims (`email`, `phone_number`,
  `address`, `name`, `family_name`, `given_name`, …) with `<redacted>`.
- `--show-pii` flag opts in to plaintext.
- Tests for both modes.

**Effort:** ~30 min.

**Status:** unblocked; small; flagged during the ID.me live test.

## Distribution

### Go and TypeScript SDKs

**Value:** multi-language coverage is table stakes for adoption in the
broader AI-agent ecosystem (most agent frameworks are split across
Python and TypeScript today). The other "agent passport" projects in
the namespace (cezexPL/agent-passport-standard) already offer multi-
language SDKs; we should match.

**Approach:** keep this Python implementation as the reference; auto-
generate or hand-write Go + TypeScript bindings against the same claim
schema and verifier semantics. Test interop with cross-implementation
fixtures.

**Status:** stretch goal; significant effort; should not start until
the Python implementation hits 1.0 and the wire format is frozen.

## Mature → 1.0 prep (when we choose to commit)

The current 0.1.x line is alpha by SemVer convention. Reaching 1.0
requires:

1. **Wire-format freeze** — pin the claim schema; document any extension
   points that adopters can rely on not breaking.
2. **Deprecation discipline** — every breaking change after 1.0 needs at
   least one minor-version notice per the [Versioning policy](https://github.com/antspriggs/nist-agent-passport/blob/main/README.md#versioning--deprecation-policy).
3. **Adoption signal** — at least one external user / integration / test
   confirming the API is usable as documented.
4. **Security review** — an external reviewer (NIST/NCCOE, OpenID
   Foundation, an OSS-security org) walks the library against its
   threat model.

**Status:** premature to schedule. We get there by working through the
strategic + library items above and seeing what real-world feedback
arrives.

## How to contribute to any of these

Pick one, open an issue describing your intended approach, then a PR
per [CONTRIBUTING.md](https://github.com/antspriggs/nist-agent-passport/blob/main/CONTRIBUTING.md).
For larger items (revocation, JWKS hosting, second SDK), discuss in
the issue before writing code — the design space matters more than
the implementation.
