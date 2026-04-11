---
type: decision
question: "How should we handle authentication?"
chosen: "JWT with 24-hour expiry, refresh tokens in HttpOnly cookies"
rejected: ["Auth0 (pricing doesn't scale)", "Roll-our-own sessions (too much maintenance)"]
rationale: "Stateless auth needed for mobile app. JWT keeps API servers simple. HttpOnly cookies prevent XSS on refresh tokens."
scope: "All API services"
status: accepted
created: 2026-04-11
last_verified: 2026-04-11
triggers: [auth, login, signup, JWT, sessions, OAuth, passwords, identity]
sources: ["mempalace://wing_project/room_auth/drawer_eval_2026", "raw/decisions/auth-evaluation.md"]
---

# Auth Approach

We use **JWT with 24-hour expiry** for all API authentication. Refresh tokens are stored in HttpOnly cookies to prevent XSS.

## Rationale
The mobile app requires stateless authentication. JWT keeps API servers simple — no session store needed. The 24-hour expiry balances security with user experience.

## Rejected Alternatives
- **Auth0**: Evaluated 2026-01. Rejected — pricing model doesn't scale past 10K MAU for our usage pattern. Source: mempalace://wing_project/room_auth/drawer_eval_2026
- **Server-side sessions**: Considered 2025-11. Rejected — requires shared session store across services, adds operational complexity we don't need yet.

<!-- 
TEMPLATE NOTE: Delete this article and create your own decisions.
This is an example showing the expected format.
-->
