# RBAC and Permissions

## Authority Rules

- Runtime authority comes from authenticated identity, active server-side tenant membership, and role-template permissions assigned through tenant membership.
- Frontend visibility is not authorization.
- New backend actions must be enforced server-side and seeded through forward-only migrations before public API exposure.

## Delivery Operations Permission Catalogue

`MVP-1.3-DELIVERY-OPERATIONS-CONTRACT-001-R2` freezes the Delivery permission names for upcoming MVP 1.3 implementation. They are not yet seeded in `app_permission`, assigned to roles, exposed through a runtime permission-catalogue bean, or enforced by public REST endpoints in the foundation slice.

| Permission | Purpose | Implementation status |
| :--- | :--- | :--- |
| `DELIVERY_VIEW` | View Delivery records and references | FROZEN_NOT_SEEDED |
| `DELIVERY_CREATE` | Create Delivery Orders | FROZEN_NOT_SEEDED |
| `DELIVERY_UPDATE` | Update editable Delivery details | FROZEN_NOT_SEEDED |
| `DELIVERY_ASSIGN` | Assign delivery resources/routes/trips | FROZEN_NOT_SEEDED |
| `DELIVERY_EXECUTE` | Execute delivery progress commands | FROZEN_NOT_SEEDED |
| `DELIVERY_POD_CAPTURE` | Capture electronic proof of delivery | FROZEN_NOT_SEEDED |
| `DELIVERY_COMPLETE` | Complete successful delivery | FROZEN_NOT_SEEDED |
| `DELIVERY_FAIL` | Record delivery failure | FROZEN_NOT_SEEDED |
| `DELIVERY_REDELIVER` | Schedule or manage redelivery | FROZEN_NOT_SEEDED |
| `DELIVERY_EXCEPTION_MANAGE` | Manage Delivery exceptions | FROZEN_NOT_SEEDED |
| `DELIVERY_REPORT_VIEW` | View Delivery reports | FROZEN_NOT_SEEDED |

Future Delivery APIs must seed and enforce the narrowest applicable permission before marking any US-56 through US-62 story implemented.
