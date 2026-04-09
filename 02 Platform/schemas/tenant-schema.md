# schema: tenant

## purpose
defines the fields that describe a tenant on the platform. a tenant is the boundary for identity, permissions, data isolation, and billing.

## fields

| field | type | description |
|---|---|---|
| `tenant_id` | string (uuid) | unique identifier, generated at creation |
| `name` | string | display name for the tenant |
| `niche` | string (enum) | which niche this tenant is using (e.g. `sports-analytics`) |
| `tier` | string (enum) | `free`, `starter`, `pro` |
| `stripe_customer_id` | string | stripe customer identifier — source of truth for billing status |
| `stripe_subscription_id` | string | active stripe subscription identifier |
| `subscription_status` | string (enum) | `active`, `past_due`, `canceled`, `trialing` |
| `config` | object | niche-specific configuration — see niche-config-schema |
| `delivery` | object | delivery channel settings — see delivery fields below |
| `created_at` | timestamp | ISO 8601 |
| `updated_at` | timestamp | ISO 8601 |

## delivery fields

| field | type | description |
|---|---|---|
| `delivery.channel` | string (enum) | `email`, `slack`, `webhook` |
| `delivery.email` | string | email address if channel is `email` |
| `delivery.slack_webhook_url` | string | slack incoming webhook url if channel is `slack` |
| `delivery.webhook_url` | string | generic webhook url if channel is `webhook` |

## rules
- `stripe_customer_id` must exist before any paid feature is accessible
- `subscription_status` is read from stripe at runtime — never cache it as the source of truth
- `niche` is set at tenant creation and does not change
- `tier` is derived from the active stripe subscription, not stored independently
- `config` contents are validated against the niche-config-schema for the tenant's niche

## what is not in this schema
- user accounts or auth (separate concern)
- agent execution history (belongs in observability layer)
- billing amounts or pricing (owned by stripe)
