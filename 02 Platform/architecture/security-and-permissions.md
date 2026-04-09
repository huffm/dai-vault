# security and permissions

Agents should never have implicit power.

All capabilities should be:
- explicitly granted
- scoped per tenant
- observable
- rate-limited where needed

Cross-tenant data access should never occur silently.
