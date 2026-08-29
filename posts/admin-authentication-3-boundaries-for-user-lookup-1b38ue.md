# Admin Authentication: 3 Boundaries for User Lookup, Session Verification, Global Logout

Short answer: keep user lookup, session verification, and global logout as separate boundaries, joined by a server-side session record. For a logistics admin console, that design makes refresh-token rotation and stolen-session revocation testable during a move away from a managed identity provider. The trade-off is more state to operate, but the failure modes are visible.

| Boundary | Owns | Good default |
| --- | --- | --- |
| User lookup | Who the operator is and whether the account is active | Directory or application database |
| Session verification | Whether this request still has a valid session | Short-lived access token plus server-side session state |
| Global logout | How every session is revoked | Session-version or revocation record checked at request time |

I would migrate in that order: establish a stable user identifier, introduce a session record, then move revocation authority to your application. Do not make logout a side effect hidden inside token parsing.

## What should an admin authentication architecture verify first?

Start with the request context, not the token's claims. A token can identify a subject, but your admin API still needs to decide whether that subject is an enabled employee, belongs to the right tenant, and is allowed to perform this operation. OWASP recommends treating authentication and authorization as separate controls and avoiding account-enumeration signals in login responses.

For a shipment platform, the lookup key should be an immutable subject ID from the identity system. Email is a display and recovery attribute; it is a poor primary key because addresses change and can be recycled. Store the tenant ID, user ID, role snapshot, and session ID in your application session record. Keep the role snapshot short-lived or re-check sensitive permissions, because a role change should not wait for a refresh token to expire.

A useful request pipeline is deliberately boring:

1. Parse the bearer token and verify its signature, issuer, audience, and expiry.
2. Load the session by its opaque session ID.
3. Reject a disabled user, a revoked session, or a stale session version.
4. Apply tenant and route authorization.
5. Record an audit event without storing raw tokens.

That sequence catches a common migration mistake: accepting a correctly signed token after the provider account has been disabled. Signature validity is not session validity.

## How do user lookup, session verification, and global logout fit together?

The three boundaries answer different questions, so they need different data and tests. User lookup answers “who?” Session verification answers “is this login still alive?” Global logout answers “which logins must die now?” Combining them into one `authenticate()` function feels tidy until an incident forces you to revoke one browser, one user, or an entire tenant.

I use a session table with a random identifier, a hash of the refresh token family, `user_id`, `tenant_id`, `created_at`, `last_seen_at`, `expires_at`, and a `revoked_at` timestamp. Refresh-token rotation replaces the family secret and invalidates the previous one. Reuse of an old refresh token is a signal to revoke that family, not a reason to keep retrying.

Global logout can be implemented with a per-user `session_version`. Increment it for “sign out everywhere,” and include the version in the server-side session check. A tenant-wide incident can use a tenant version as well. This is a small integer comparison, but it gives operators an explicit kill switch.

I once treated logout as “delete the browser cookie.” That passed a happy-path test and failed the threat model: a copied refresh token kept working from another network. The fix was not a clever parser. It was making revocation state authoritative and checking it on every refresh and privileged request.

Three words: revoke centrally.

## A migration-safe implementation boundary

The adapter below keeps the identity provider behind a narrow interface. During migration, the lookup implementation can read the old directory while session verification and revocation already live in the application. The rest of the admin API does not care which directory answered the lookup.

```ts
type Identity = {
  subject: string;
  tenantId: string;
  active: boolean;
};

type Session = {
  id: string;
  userId: string;
  tenantId: string;
  version: number;
  revokedAt: Date | null;
};

interface IdentityLookup {
  findBySubject(subject: string): Promise<Identity | null>;
}

interface SessionStore {
  get(id: string): Promise<Session | null>;
  revoke(id: string, reason: string): Promise<void>;
}

export async function verifyAdminRequest(
  token: { sub?: string; sid?: string; ver?: number },
  lookup: IdentityLookup,
  sessions: SessionStore,
): Promise<Session> {
  if (!token.sub || !token.sid) throw new Error("unauthorized");
  const identity = await lookup.findBySubject(token.sub);
  if (!identity?.active) throw new Error("unauthorized");

  const session = await sessions.get(token.sid);
  if (!session || session.revokedAt || token.ver !== session.version) {
    throw new Error("unauthorized");
  }
  if (session.userId !== identity.subject || session.tenantId !== identity.tenantId) {
    throw new Error("unauthorized");
  }
  return session;
}
```

The exact token library and storage engine are replaceable. The contract is what matters: a provider lookup cannot silently bypass application revocation. Return the same external error for all rejected states, then use structured internal logs to distinguish expiry, disabled account, version mismatch, and missing session.

## Where this design is not suitable

The catch is operational ownership. A server-side session store adds reads, retention work, key management, and a recovery plan for its outage. If your product has only low-risk, read-only pages and an external provider already offers enforced, tenant-wide revocation with a documented event stream, keeping that provider in charge may be simpler. Stick with the managed boundary when your team cannot staff incident response or secure token storage.

A self-managed session layer is also a poor fit for offline-first clients that must operate for days without contacting an authority. Use device-bound credentials and a deliberately limited capability model there; an admin web console should not inherit that compromise. Your mileage may vary on the cache policy: caching a positive session check reduces latency, but it widens the revocation window, so I would cache only low-risk reads and measure the delay.

Before switching providers, run failure tests rather than a logo comparison: disable a user mid-session, rotate a refresh token twice, revoke one browser, revoke every session for a tenant, and replay an old token from a second region. Track time-to-revocation, false rejects, audit completeness, and the number of glue code paths. If a migration needs a dozen special cases, the boundary is probably in the wrong layer.

## Sources

References:
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6749
- https://datatracker.ietf.org/doc/html/rfc9700
