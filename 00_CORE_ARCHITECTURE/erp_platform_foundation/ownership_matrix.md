# Current → Target Ownership Matrix

Proposed owners are recommendations only unless status says otherwise. No row authorizes migration.

| Concept | Current owner | Evidence | Proposed target owner | Consumers | Conflict | Reconciliation | Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| Organization | Organization context plus Platform target registry | Current Transportation/Organization KB and Platform ADR | Platform Management for subscribing organization; customer relationship separate | All | Name collision between tenant organization and CRM account | Map concepts; never merge by label | DECISION REQUIRED |
| Tenant | Tenancy/Identity authority | Multi-tenancy standards; current Transport tenancy | Platform tenant authority coordinated with Control Plane | All | Authority handoff | Staged trust contract and stable UUID | DECISION REQUIRED |
| User Account | Transport Identity | Identity/RBAC implementation evidence | Shared Platform Identity | All | Existing JWT/account IDs | Staged migration/adapters; preserve audit | DECISION REQUIRED |
| Role | Transport Identity | RBAC registry | Platform Identity, tenant-scoped | All | Role semantic drift | Map permissions, not role names | DECISION REQUIRED |
| Permission | Owning application Identity catalogue | RBAC registry | Shared governance with domain-owned actions | All | Duplicate action codes | Inventory and compatibility mapping | DECISION REQUIRED |
| Subscription | Platform target only | Platform ADR/policy | Platform Management | Customer ERP | Not implemented | Approve D-ERP-15 contracts | DECISION REQUIRED |
| Industry Entitlement | Platform target only | Platform ADR | Platform Management | Frontend/runtime | Industry may be confused with access | Separate classification/defaults from entitlement | DECISION REQUIRED |
| Employee | No enterprise HRM; Driver has employee/contact facts | Transportation + HRM blueprint | HRM | Transport, Projects, Maintenance | Driver overlap | Stable logical Employee mapping | DECISION REQUIRED |
| Employment | No enterprise owner | HRM blueprint | HRM | Transport, Finance, Projects | Identity/account conflation | Keep Account distinct; effective-dated bridge | DECISION REQUIRED |
| Driver | Transportation Fleet/Driver | Transportation module | Transportation operational Driver | HRM, Trip, Delivery | Workforce versus safety facts | Split after record classification | DECISION REQUIRED |
| Driver Qualification | Transportation | Licence/medical/drug/exception records | Split HRM general qualification vs Transport eligibility | Transport, Compliance | Privacy and timing | Minimized facts; no record copying | DECISION REQUIRED |
| Driver Availability | Transportation | Driver availability/Trip assignments | Transportation operational projection consuming HRM | Trip, Delivery | Cross-domain timing | Synchronous validation plus change/reconcile | DECISION REQUIRED |
| Customer | Organization context | Customer APIs/Delivery contact use | Sales/CRM canonical Account relationship | Transport, Finance | Existing UUID/API/contact authority | Mapping/compatibility; preserve token privacy | DECISION REQUIRED |
| Supplier | Transport vendor/Fuel concepts | Procurement blueprint legacy note | Procurement or separately approved shared master | Procurement, Finance, Maintenance | Provider/vendor/supplier semantics | Classify and map; preserve Fuel APIs | DECISION REQUIRED |
| Project | Transportation | Trip project_id and project rows | Project Management | Transport, Finance, Procurement | Physical legacy ownership | Stable UUID map and validation port | DECISION REQUIRED |
| Vehicle | Transportation Fleet | Vehicle aggregate | Fleet/Transportation | Trip, Maintenance, Tracking | Maintenance may duplicate identity | Retain Vehicle ID/master | DECISION REQUIRED |
| Vehicle Allocation | Trip | Trip assignment facts | Transportation Trip | Fleet, Tracking | May be mistaken for Fleet ownership | Keep assignment fact; publish projection | DECISION REQUIRED |
| Vehicle Availability | Fleet/Trip plus maintenance_schedule | Current assignment/maintenance checks | Transportation composition of Fleet + Maintenance facts | Trip, Delivery | Safety/consistency | Compatibility bridge then explicit eligibility contract | DECISION REQUIRED |
| Vehicle Maintenance | Transportation maintenance_schedule | Fleet linkage | Vehicle Maintenance | Transport, Finance | Existing blocking behavior | Characterize and stage ownership migration | DECISION REQUIRED |
| Maintenance Work Order | Not implemented as enterprise capability | Maintenance blueprint | Vehicle Maintenance | Inventory, Finance | No current owner | Approve domain before implementation | DECISION REQUIRED |
| Item/Product | Fuel-specific products; no enterprise item master | Inventory blueprint | Inventory | Procurement, Maintenance, Sales | Specialized fuel versus general item | Explicit classifications/mapping | DECISION REQUIRED |
| Warehouse | No enterprise owner | Inventory blueprint | Inventory | Procurement, Projects, Transport | None implemented | Approve before build | DECISION REQUIRED |
| Stock | Fuel bunker ledger is specialized current owner | Transport Fuel evidence | Inventory for general stock; Fuel for bunker exception | Maintenance, Procurement | Duplicate ledgers | Do not migrate by name; define boundaries | DECISION REQUIRED |
| Stock Reservation | No enterprise owner | Inventory blueprint | Inventory | Maintenance, Sales, Projects | Distributed reservation | Idempotent command and expiry/reconcile | DECISION REQUIRED |
| Purchase Requisition | No enterprise owner | Procurement blueprint | Procurement | Projects, Maintenance, Inventory | Generic workflow temptation | Domain-specific lifecycle first | DECISION REQUIRED |
| Purchase Order | No enterprise owner | Procurement blueprint | Procurement | Inventory, Finance | Fuel purchase overlap | Compatibility decision | DECISION REQUIRED |
| Goods Receipt | No enterprise owner | Inventory blueprint | Inventory | Procurement, Finance | Three-way match boundary | Local receipt then durable facts | DECISION REQUIRED |
| Sales Lead | No enterprise owner | Sales/CRM blueprint | Sales/CRM | Reporting | None | Approve scope | DECISION REQUIRED |
| Opportunity | No enterprise owner | Sales/CRM blueprint | Sales/CRM | Finance/Reporting | None | Approve scope | DECISION REQUIRED |
| Sales Order | No enterprise owner | Sales/CRM blueprint | Sales/CRM | Inventory, Transport, Finance | Transport order confusion | Explicit logical linkage | DECISION REQUIRED |
| Transport Order | Freight/Delivery | Transportation module | Transportation | Sales/CRM, Billing | Overlap with Sales Order | Reference, never reuse aggregate | DECISION REQUIRED |
| Trip | Transportation Trip | Implemented aggregate | Transportation | Tracking, Billing, Finance | None | Preserve owner and publish minimized facts | CURRENT VERIFIED IMPLEMENTATION |
| Route | Transportation Routing | Implemented aggregate | Transportation | Trip, Tracking | None | Preserve owner | CURRENT VERIFIED IMPLEMENTATION |
| Delivery | Transportation Delivery | US56–70 evidence | Transportation | Sales/CRM, Finance | Sales fulfillment overlap | Keep operational Delivery; link order/customer | CURRENT VERIFIED IMPLEMENTATION |
| Fuel Issue | Transportation Fuel | Implemented lifecycle/ledger | Transportation | Finance, Compliance | Inventory naming overlap | Preserve operational Fuel owner | CURRENT VERIFIED IMPLEMENTATION |
| Transport Charge | Transport Billing | US-47 accepted boundary | Transport Billing | Finance | Could be confused with invoice | Export immutable source fact | ACCEPTED EXISTING DECISION |
| Customer Invoice | No implemented Finance owner; Transport bill is not invoice | Finance blueprint/US-47 ADR | Finance | Sales, Transport | Terminology collision | Keep operational bill distinct | DECISION REQUIRED |
| Accounts Receivable | No current enterprise owner | Finance blueprint | Finance | Sales, Transport | None | Approve Finance boundary | DECISION REQUIRED |
| Accounts Payable | No current enterprise owner | Finance blueprint | Finance | Procurement | None | Approve Finance boundary | DECISION REQUIRED |
| General Ledger | No current enterprise owner | Finance blueprint | Finance | All posting sources | None | Approve Finance boundary | DECISION REQUIRED |
| Payment | Transport source/payment-like operational facts; SaaS payment target separate | Finance and Platform policies | Finance for customer accounting; Platform for SaaS collection | Many | Same word, different obligations | Explicit bounded identities/contracts | DECISION REQUIRED |
| Document | Fleet/Delivery/Compliance domain evidence; no generic file owner | Current module docs | Owning domain metadata; specialized file service candidate | All | Duplicate generic document master | Classify content/metadata/retention first | DECISION REQUIRED |
| Notification | Transportation Notification | Implemented Notification context | Notification capability | All | Recipient/source ownership | Minimized facts; owner retains meaning | CURRENT VERIFIED IMPLEMENTATION |
| Audit | Per-owner immutable evidence plus technical audit | Module schemas/ADRs | Each domain owns audit; platform-wide view is read model | All | Central mutable audit ambiguity | Standard envelope/read model, no foreign writes | DECISION REQUIRED |
| Compliance | Compliance context | V108+ current inactive capability | Compliance decision context | Fleet, Driver, Freight, Billing | Policy authority pending | Keep default-off and fact contracts | CURRENT VERIFIED IMPLEMENTATION |
| Tracking/Telemetry | Tracking context | Accepted ADR and Kafka/Redis/Timescale evidence | Tracking | Fleet, Trip, Routing, Ops | Specialized infrastructure | Preserve workload boundary | CURRENT VERIFIED IMPLEMENTATION |
| Reporting/Analytics | Domain analytics + Reporting composition | Fuel/Delivery/Tracking records | Domain interpretations; governed composition/read models | All | Metric redefinition/foreign reads | Lineage and owner-produced facts | DECISION REQUIRED |
