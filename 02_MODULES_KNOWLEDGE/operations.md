# Operations Module Knowledge Base

## 1. Domain Overview
The `operations` module owns operational exception management (US-78) and operational disruption coordination & impact evaluation (US-86).

---

## 2. Database Schema

### Table: `operational_disruption` (Flyway V110)
| Column | Type | Nullable | Constraints & Description |
|---|---|---|---|
| `id` | UUID | NO | Primary Key |
| `tenant_id` | UUID | NO | Tenant isolation identifier |
| `disruption_reference` | VARCHAR(32) | NO | Unique reference (`ODIS-...`) |
| `title` | VARCHAR(120) | NO | Disruption headline title |
| `description` | VARCHAR(2000) | YES | Details and description |
| `disruption_type` | VARCHAR(32) | NO | `WEATHER`, `CIVIL_RESTRICTION`, `STRIKE`, `BORDER_RESTRICTION`, `DEMAND_SPIKE`, `NATURAL_DISASTER`, `ROAD_BLOCKAGE` |
| `severity` | VARCHAR(16) | NO | `LOW`, `MEDIUM`, `HIGH`, `CRITICAL` |
| `status` | VARCHAR(24) | NO | `DECLARED`, `ASSESSING_IMPACT`, `IMPACT_ASSESSED`, `REPLAN_PROPOSED`, `RESOLVED` |
| `geographic_zone` | VARCHAR(80) | NO | Geographic region or zone |
| `start_time` | TIMESTAMPTZ | NO | Disruption start timestamp |
| `estimated_end_time` | TIMESTAMPTZ | YES | Estimated end timestamp (>= startTime) |
| `actual_end_time` | TIMESTAMPTZ | YES | Actual resolution timestamp |
| `constraint_rules` | TEXT | YES | Semicolon-delimited rule definitions |
| `affected_trip_count` | INT | NO | Default 0 |
| `affected_trip_ids` | TEXT | YES | Comma-delimited list of affected active trips |
| `affected_route_ids` | TEXT | YES | Comma-delimited list of affected routes |
| `affected_vehicle_ids` | TEXT | YES | Comma-delimited list of affected vehicles |
| `affected_driver_ids` | TEXT | YES | Comma-delimited list of affected drivers |
| `replanning_recommended` | BOOLEAN | NO | Replanning recommendation flag |
| `recommended_actions` | TEXT | YES | Semicolon-delimited recommended actions |
| `impact_summary_notes` | VARCHAR(2000) | YES | Assessment summary notes |
| `assessed_at` | TIMESTAMPTZ | YES | Timestamp of impact assessment |
| `resolved_by` | UUID | YES | User ID resolving the disruption |
| `resolution_notes` | VARCHAR(2000) | YES | Resolution explanation |
| `version` | BIGINT | NO | Optimistic lock version |
| `created_at` | TIMESTAMPTZ | NO | Record creation timestamp |
| `updated_at` | TIMESTAMPTZ | NO | Record update timestamp |

---

## 3. Integration & Domain Events
- `OperationalDisruptionDeclaredEvent` (`OPERATIONAL_DISRUPTION_DECLARED_V1`): Published upon disruption declaration.
- `OperationalDisruptionResolvedEvent` (`OPERATIONAL_DISRUPTION_RESOLVED_V1`): Published upon resolution.

---

## 4. REST API & RBAC
- `POST /api/v1/operations/disruptions`: Authority `OPERATIONAL_EXCEPTION_MANAGE`
- `POST /api/v1/operations/disruptions/{id}/assess-impact`: Authority `OPERATIONAL_EXCEPTION_MANAGE`
- `POST /api/v1/operations/disruptions/{id}/resolve`: Authority `OPERATIONAL_EXCEPTION_MANAGE`
- `GET /api/v1/operations/disruptions/{id}`: Authority `OPERATIONAL_EXCEPTION_VIEW`
- `GET /api/v1/operations/disruptions`: Authority `OPERATIONAL_EXCEPTION_VIEW`
