# ADR 0003: Authentication Token Handling Strategy

## Status
Proposed

## Context
The Cario Trace frontend is a React SPA frontend with a GraphQL Gateway and Core Backend (Django DRF). Users authenticate via Auth0.  

After login, the frontend must store and use authentication tokens to communicate with backend services. Key considerations include:
- Security (XSS/CSRF protection)
- Simplicity for a portfolio/demo project
- Statelessness of backend
- Compatibility with SPA architecture

## Decision
The token handling approach has not yet been finalized. Two options are under consideration:

1. **Bearer token in frontend memory or localStorage**
   - Frontend includes `Authorization: Bearer <token>` in requests
   - Simple, no CSRF, minimal backend complexity
   - Exposed to XSS if frontend is compromised

2. **Secure HttpOnly cookie**
   - Cookie set by Auth Service with `HttpOnly`, `Secure`, `SameSite=Lax`
   - Browser sends cookie automatically
   - Protected from XSS, but requires CSRF and CORS handling

The final choice will be decided after evaluating implementation complexity and security requirements.

## Consequences
- Frontend implementation depends on chosen approach (Authorization header vs cookie)
- Gateway and backend may need minor adjustments (extract token from header or cookie)
- Security posture differs:
  - Bearer token → higher XSS risk
  - Cookie → need CSRF protection
- Both options preserve backend statelessness