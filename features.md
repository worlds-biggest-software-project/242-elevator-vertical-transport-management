# Elevator & Vertical Transport Management — Feature & Functionality Survey

> Candidate #242 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| LiftNet | Vertical transport management + remote control | Commercial SaaS | https://liftnet.com/ |
| iFactory Elevator Module | Compliance lifecycle + IoT monitoring | Commercial SaaS | https://ifactoryapp.com/ |
| BRYCER (The Compliance Engine) | AHJ-side compliance tracking | Commercial SaaS | https://www.thecomplianceengine.com/ |
| FIELDBOSS | Elevator-specialist FSM on Microsoft Dynamics 365 | Commercial SaaS | https://www.fieldboss.com/ |
| ElevatorPlus | Mobile-first elevator contractor management | Commercial SaaS | https://www.elevatorplus.app/ |
| Fieldy | Field service management for elevator contractors | Commercial SaaS | https://getfieldy.com/ |
| ServiceTitan | Full-suite commercial field service ERP | Commercial SaaS | https://www.servicetitan.com/ |
| SafetyCulture (iAuditor) | Digital inspection and audit platform | Commercial SaaS (freemium) | https://safetyculture.com/ |
| UpKeep | Mobile CMMS for facilities teams | Commercial SaaS | https://upkeep.com/ |
| Simpro | Field service ERP with digital forms | Commercial SaaS | https://www.simprogroup.com/ |
| Otis ONE | OEM remote monitoring and connected services | Commercial SaaS (OEM) | https://www.otis.com/ |
| KONE 24/7 Connected Services | OEM predictive maintenance and IoT monitoring | Commercial SaaS (OEM) | https://www.kone.com/ |

---

## Feature Analysis by Solution

### LiftNet

**Core features**
- Real-time fault logging with early shutdown detection and operational bottleneck identification
- Live dashboard with customisable metrics and unit-status displays
- Traffic and usage analysis for building efficiency optimisation
- Unit control: floor lockouts, independent service, hall calls, car-to-lobby, fire service, and Code Blue
- Predictive analytics for downtime minimisation and service-call optimisation
- Multi-brand compatibility through a single virtual portal, regardless of equipment manufacturer
- Cloud-hosted with automated updates and no on-premise infrastructure required
- Group management for multi-unit buildings

**Differentiating features**
- Hardware-plus-software bundle: integrates with DST controller hardware for deep signal-level control
- Supports all three major vertical transport equipment types: elevators, escalators, and moving walkways
- Code Blue emergency call integration and immediate entrapment response workflow
- Instant remote unit isolation without dispatching a technician

**UX patterns**
- Customisable live dashboard surfacing the most actionable metrics
- Alert-driven interface focused on faults that require immediate attention
- Consumable summary reports for building managers

**Integration points**
- DST hardware controllers (native)
- Third-party elevator controllers via hardware interface layer
- Cloud-based portal accessible via web browser

**Known gaps**
- No AHJ-facing compliance portal
- Limited native compliance calendar or permit-tracking features
- No technician job dispatch or field service workflow
- No open API or BMS data-feed integration advertised publicly

**Licence / IP notes**
- Proprietary commercial SaaS; pricing by custom quote. No open-source components identified.

---

### iFactory Elevator Module

**Core features**
- Jurisdiction-aware compliance calendar auto-generated from state/city + ASME A17.1 adoption database
- Automated permit filing to city/state departments with digital operating permit issuance
- Safety gate: blocks certificate filing when sensor data (e.g. brake deceleration) falls outside A17.1 safety window
- AI-powered deadline alert engine dispatching 60-day advance notifications for Cat 1/Cat 5 tests
- Public Inspection Portal for authorised state auditors — blockchain-verified, unalterable records
- IoT retrofit sensor mesh for OEM-agnostic real-time telemetry collection
- CMMS integration for automated technician dispatch

**Differentiating features**
- Only solution combining IoT telemetry with code-version-aware compliance validation
- Blockchain-verified audit trail claimed to reduce physical audit time by 75%
- Local code modification awareness (e.g. NYC Appendix K, Chicago amendments) applied automatically on unit onboarding
- Safety gate that prevents filing non-compliant certificates regardless of user override

**UX patterns**
- Onboarding wizard auto-configures compliance requirements from location + equipment data
- Compliance calendar view with colour-coded deadline urgency
- Auditor-facing read-only portal requiring no software license

**Integration points**
- OEM-agnostic IoT sensor layer (Schindler, ThyssenKrupp, Otis retrofittable)
- CMMS dispatch integrations
- State/city permit authority digital filing endpoints

**Known gaps**
- Primarily North American (ASME A17.1); limited EN 81 / international code coverage stated
- Sensor retrofit costs not publicly disclosed; may be prohibitive for smaller fleets
- No publicly documented open API for third-party BMS integration

**Licence / IP notes**
- Proprietary commercial SaaS; custom pricing. Blockchain-verification component may involve third-party IP.

---

### BRYCER (The Compliance Engine)

**Core features**
- Centralised compliance tracking for fire protection, backflow, and elevator inspections under one platform
- Annual and periodic elevator inspection tracking with deficiency classification by severity
- Automated outreach workflows to property owners and contractors when inspections are due or deficiencies remain unresolved
- Report validation by BRYCER-certified fire protection professionals embedded in the submission workflow
- Full digital audit trail for AHJ decision-making
- Notifications configurable per jurisdiction; communications branded per AHJ
- Integration with field service platforms (e.g. ServiceTrade) for automated AHJ reporting

**Differentiating features**
- Designed specifically for AHJ workflows — not contractor-facing
- Human expert review layer on every submitted report (certified reviewers verify accuracy and deficiency classification)
- Adopted by over 1,420 AHJs nationwide, creating a large network effect
- Supports backflow + fire + elevator in one platform, enabling cross-system compliance view for jurisdictions
- Implementation completed in 60 days with included training and customer success

**UX patterns**
- AHJ-facing dashboard with actionable insights and automated outreach status
- Property and contractor portals with role-based access
- Virtual walkthrough onboarding service to reduce setup friction

**Integration points**
- ServiceTrade integration for contractor-side AHJ report automation
- Configurable outreach via email/SMS to property owners
- Branded AHJ portals with jurisdiction-level configuration

**Known gaps**
- Not a maintenance management or contractor dispatch platform
- Elevator capability appears secondary to fire/life safety primary use case
- No IoT/telemetry ingestion or predictive maintenance features
- North American focus; no evidence of international jurisdiction support

**Licence / IP notes**
- Proprietary commercial SaaS; custom AHJ-focused pricing. Human review layer is a proprietary service differentiator.

---

### FIELDBOSS

**Core features**
- Built on Microsoft Dynamics 365, delivering project and service management in a single system
- Separate tracking of installation projects vs. maintenance service work with independent P&L visibility
- Work order management, job costing, dispatching, and equipment history in one system
- Integration with Dynamics 365 Finance and Operations for accounting and payroll
- Mobile app for field technicians
- Multi-location and multi-division reporting with roll-up analytics
- Customer and contract management (AMC and project billing)

**Differentiating features**
- Only purpose-built elevator FSM solution built natively on Microsoft Dynamics 365 ecosystem
- Tracks distinct installation vs. maintenance profitability — uniquely valuable for contractors managing both
- Enterprise-grade ERP capabilities (project costing, multi-entity consolidation) not found in lighter FSM tools
- Direct integration with Dynamics 365 Supply Chain for parts and inventory

**UX patterns**
- ERP-style interface familiar to Dynamics users; deep configuration expected before go-live
- Separate operational views for project managers vs. field dispatchers vs. executives
- Role-based dashboards with drill-down from portfolio to job level

**Integration points**
- Microsoft Dynamics 365 Finance, Supply Chain, and CRM (native)
- Microsoft Power BI for reporting
- Open APIs via Dynamics 365 platform
- Third-party accounting (via Dynamics connectors)

**Known gaps**
- Heavy implementation effort; not suitable for small contractors
- Compliance calendar and permit management not highlighted as native features
- No IoT/sensor integration or predictive maintenance capabilities
- Pricing not published; likely high relative to lighter SaaS alternatives

**Licence / IP notes**
- Proprietary commercial SaaS built on Microsoft Dynamics 365 (Microsoft EULA applies). No open-source components identified.

---

### ElevatorPlus

**Core features**
- QR-code-enabled breakdown reporting: customers scan a code to report faults, triggering automatic technician assignment
- Instant breakdown notifications to managers with response tracking
- AMC renewal reminders and recurring maintenance scheduling
- Real-time GPS tracking of all field employees
- Quotation generation and invoicing
- Mobile app for technicians (works offline with auto-sync)
- Job history and service logs per unit

**Differentiating features**
- QR-code-to-breakdown workflow removes phone call friction from fault reporting
- Designed specifically for the elevator AMC (Annual Maintenance Contract) business model common in South/Southeast Asia
- Offline-capable mobile app suited to low-connectivity building environments

**UX patterns**
- Customer-facing QR code generates a fault ticket without any app download required
- Technician app designed for minimal training; task-list driven
- Manager dashboard focused on breakdown response KPIs

**Integration points**
- Mobile app (iOS/Android)
- QR code generation and scanning
- Limited public API documentation identified

**Known gaps**
- No compliance calendar or regulatory permit management
- Limited analytics or predictive maintenance capability
- No BMS or IoT sensor integration
- Primarily Indian market focus; limited evidence of international regulatory support

**Licence / IP notes**
- Proprietary commercial SaaS (Indian market). No open-source components identified.

---

### Fieldy

**Core features**
- AMC automation: auto-renewal reminders, recurring job scheduling, smart auto-assignment
- GPS-tracked technician dispatch with real-time movement visibility
- Mobile app with offline capability and automatic data sync on reconnect
- Job scheduling, invoicing, and quote generation
- Instant alerts on major job updates to office and customer
- Field crew performance tracking
- Customer portal for service request submission

**Differentiating features**
- Designed around the AMC subscription model dominant in elevator maintenance contracts
- Ultra-lightweight mobile app (minimal battery and data consumption) for field use
- Smart auto-assignment for recurring jobs reduces dispatcher workload

**UX patterns**
- Dashboard focused on AMC coverage rates and renewal pipeline
- Mobile-first design with office web portal for managers
- Automated notification flows reduce manual follow-up

**Integration points**
- Mobile app (iOS/Android)
- SMS and in-app notifications
- Limited third-party integrations publicly documented

**Known gaps**
- No compliance regulatory calendar or permit-filing features
- No IoT/sensor integration or predictive maintenance
- No open API documented for BMS or third-party platform connections
- Reporting less customisable than enterprise CMMS solutions

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components identified.

---

### ServiceTitan

**Core features**
- Scheduling and dispatching with real-time GPS tracking
- Work order management, job costing, and multi-division tracking
- Customer relationship management and service agreement management
- Inventory management with parts tracking and first-time fix rate optimisation
- Mobile app for technicians
- Comprehensive reporting and analytics across all operational metrics
- Integration with major accounting ERP platforms (QuickBooks, Sage, Acumatica, NetSuite, Oracle)
- Open API for integration with third-party platforms

**Differentiating features**
- Broadest ecosystem of integrations of any FSM platform, including ERP and CRM
- Supports multi-trade operations (elevator, HVAC, electrical) under one account
- Real-time comprehensive reporting with roll-up across multiple locations or divisions

**UX patterns**
- Full-featured dispatch board with technician location overlay
- Customer-facing booking and communication portal
- Role-based access across executive, operations, and field views

**Integration points**
- Open REST API
- Native integrations with QuickBooks, Sage, Acumatica, NetSuite, Oracle, Foundation
- ServiceTrade integration available for AHJ compliance reporting (via BRYCER)
- Payroll and HR system connectors

**Known gaps**
- Not elevator-specific: lacks native ASME/EN 81 compliance calendar, permit filing, or unit-level shaft tracking
- Generic asset model cannot independently track compliance records per elevator shaft
- No IoT/sensor data ingestion or predictive maintenance features
- No emergency phone mapping capability

**Licence / IP notes**
- Proprietary commercial SaaS; enterprise pricing. No elevator-specific IP concerns identified.

---

### SafetyCulture (iAuditor)

**Core features**
- Digital inspection checklists and audit templates (including ASME A17.1-aligned elevator templates in template library)
- Photo and video capture during inspections with annotation
- Corrective action assignment and tracking directly from inspection findings
- Scheduling of recurring inspections with automated reminders
- Offline-capable mobile data collection
- Integration with third-party FSM and CMMS platforms

**Differentiating features**
- Largest publicly available library of inspection templates, including elevator-specific forms
- Corrective action workflow embedded directly in inspection flow (deficiency-to-action in one step)
- Supports lone worker safety check-ins for field inspectors

**UX patterns**
- Template-builder with conditional logic (skip/show questions based on prior answers)
- Mobile inspection UI designed for one-handed use in the field
- Manager dashboard showing inspection completion rates and open corrective actions

**Integration points**
- REST API and Zapier integration
- Integrations with Slack, Microsoft Teams, Power BI, Salesforce, and major CMMS platforms
- Webhook support for real-time event-driven integrations

**Known gaps**
- Not a dispatching, invoicing, or CRM tool — purely inspection and audit
- No elevator-specific compliance calendar, permit management, or regulatory filing
- No IoT/sensor integration
- Template-based approach requires manual template maintenance when codes change

**Licence / IP notes**
- Freemium to $24/seat/month; proprietary commercial SaaS. Open template library is freely shared. No elevator-specific IP concerns.

---

### UpKeep

**Core features**
- Work order creation, assignment, and tracking via mobile app
- Preventive maintenance scheduling by calendar or meter/usage triggers
- Asset management with full service history per equipment unit
- Inventory and parts management with low-stock alerts
- Photo and document attachment to work orders
- Reporting dashboards for MTTR, PM completion rate, and downtime
- Integration with BMS, ERP, and IoT sensor platforms via API

**Differentiating features**
- Strong mobile-first UX with rapid technician onboarding
- Meter-based PM triggers (e.g., cycle count for elevator door operators)
- IoT sensor integration to auto-generate work orders from threshold alerts

**UX patterns**
- Requester portal allows non-technical building staff to submit work orders without full system access
- Mobile app optimised for quick in-field status updates
- Dashboard focused on maintenance KPIs and asset reliability

**Integration points**
- REST API and Zapier
- Native IoT sensor data ingestion (threshold-based work order triggers)
- BMS integration via API
- ERP accounting integrations

**Known gaps**
- No elevator-specific compliance calendar or regulatory permit tracking
- Cannot track elevator shaft as a distinct compliance entity separate from physical location
- No AHJ-facing reporting or permit-filing workflow
- Per-user pricing scales poorly for large technician teams

**Licence / IP notes**
- From $45/user/month; proprietary commercial SaaS. No elevator-specific IP concerns identified.

---

### Otis ONE

**Core features**
- 24/7 real-time monitoring of elevator systems: cab, machine room, hoistway, and pit
- Monitoring of up to 200 critical operational parameters per unit
- Predictive maintenance analytics using IoT sensor data and historical performance
- Building Management API for exporting elevator data to BMS, BAS, and SCADA systems
- Real-time operational data (availability, maintenance insights, traffic per floor) via API
- Two-way API support enabling real-time status feedback during robot and autonomous system interactions
- Sandbox environment and extensive developer documentation via Otis Developer Portal

**Differentiating features**
- Deepest sensor coverage of any platform: 200 monitored parameters vs. generic IoT approaches
- Two-way robot/autonomous-vehicle API (elevator-to-robot and robot-to-elevator communication)
- Developer portal with sandbox environment for third-party BMS/SCADA integration development

**UX patterns**
- Building manager portal showing unit health status and maintenance forecasts
- Developer portal at developers.otis.com for API access and sandbox testing
- Maintenance insights surfaced alongside real-time operational data

**Integration points**
- Building Management API (REST; developers.otis.com)
- BMS, BAS, SCADA system integration
- Autonomous robot and delivery system integration
- Otis eService mobile app for technician use

**Known gaps**
- Locked to Otis-branded equipment only; no multi-OEM support
- No contractor dispatch, invoicing, or contract management
- No compliance calendar or permit-filing workflow
- API requires active Otis service contract; not available independently

**Licence / IP notes**
- Proprietary OEM SaaS bundled with equipment maintenance contracts. Developer portal API available but gated by Otis contract. No open-source components.

---

### KONE 24/7 Connected Services

**Core features**
- Monitoring of up to 200 critical parameters per elevator unit via IoT sensors and GSM connectivity
- AI/Watson-powered analytics identifying failure symptom patterns from millions of data points
- Remote remediation capability: significant percentage of escalator stoppages resolved remotely without dispatch
- Elevator WebSocket API v2 for real-time call execution and call/elevator status streaming
- Site Monitoring API for real-time lift status, call state, deck position, and door state
- Service order integration API for BMS/asset management system connection
- KONE Developer Portal (dev.kone.com) with TypeScript SDK examples on GitHub

**Differentiating features**
- Watson AI analytics processing millions of sensor data points for symptom identification before failure
- Remote automatic remediation (not just alerting) for a class of escalator fault conditions
- Publicly accessible Developer Portal with WebSocket API and open example code on GitHub (konecorp/kone-api-examples)

**UX patterns**
- Building manager portal showing connected equipment health and maintenance schedule
- Developer portal at dev.kone.com with API sandbox and documentation
- Service notifications push to maintenance teams and building managers

**Integration points**
- KONE Elevator Call API (WebSocket v2) — dev.kone.com
- KONE Site Monitoring API — real-time equipment telemetry
- Service Order API — integration with BMS and asset management platforms
- GitHub TypeScript examples: github.com/konecorp/kone-api-examples

**Known gaps**
- Locked to KONE-branded equipment; no multi-OEM support
- No contractor dispatch, invoicing, or compliance calendar features
- API access gated by KONE service contract
- No permit filing or AHJ-facing workflow

**Licence / IP notes**
- Proprietary OEM SaaS. KONE API example code published on GitHub (konecorp organisation); licence terms for examples available in repo. Platform API itself is proprietary and gated.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Work order creation, assignment, and status tracking
- Preventive maintenance scheduling (calendar and/or usage-triggered)
- Asset register with per-unit service history
- Mobile app for field technicians (offline capability strongly preferred)
- Customer notification of scheduled visits and job completion
- Basic reporting: open/closed work orders, PM completion rates

### Differentiating Features
- Jurisdiction-aware compliance calendar linked to specific ASME A17.1/EN 81 code versions
- Automated permit filing and digital certificate issuance
- IoT/sensor telemetry integration with anomaly detection and predictive maintenance
- QR-code-driven fault reporting (removes call-centre friction for building occupants)
- Blockchain-verified or tamper-evident audit trails for high-stakes compliance records
- AHJ-facing portals enabling regulatory oversight without requiring AHJ adoption of contractor tools
- Two-way robot/autonomous-system API integration (emerging requirement for smart building deployments)
- Installation vs. maintenance project tracking with separate P&L visibility

### Underserved Areas / Opportunities
- **Multi-OEM predictive maintenance**: OEM IoT platforms (Otis ONE, KONE 24/7) are siloed by brand; buildings with mixed-OEM fleets have no unified telemetry view
- **Shaft-level compliance entity model**: generic FSM tools track assets at equipment or location level but cannot independently manage compliance records, component logs, and inspection histories for each elevator shaft
- **Emergency phone mapping**: no generic FSM platform natively maps elevator emergency phone numbers to specific units — a regulatory requirement under ASME A17.1
- **International code management**: most compliance tools cover ASME A17.1 (North America) only; no tool bridges A17.1, EN 81, and emerging ISO 8100 global harmonisation
- **Regulatory change monitoring**: no platform actively monitors code amendment publications and flags which fleet units require remedial action
- **AI-assisted natural-language incident reporting**: technicians still manually type or dictate notes; structured defect classification is not automated

### AI-Augmentation Candidates
- Anomaly detection on raw sensor streams (motor current, door timing, vibration, brake deceleration) to generate predictive maintenance alerts ahead of breakdown
- NLP-based incident report parsing to auto-classify defect type, component, severity, and recommended corrective action
- Portfolio risk scoring across large building fleets using age, usage cycles, deficiency history, and local code compliance status
- Automated regulatory change monitoring with AI-driven impact assessment: which units need action when a new ASME A17.1 amendment is published
- AI deadline engine that cross-references each unit's specific jurisdiction, equipment type, and code version to generate individualised compliance calendars

---

## Legal & IP Summary

No elevator-specific patents were identified in the course of this research. The primary proprietary differentiators across vendors — blockchain-verified records (iFactory), Watson AI analytics (KONE), the 200-parameter monitoring architecture (Otis/KONE), and the Microsoft Dynamics 365 platform integration (FIELDBOSS) — are either trade-secret implementations or rely on third-party platform licences rather than patents filed against feature concepts. Open-source components are scarce in this space; the most notable exception is KONE's public GitHub repository of API example code (konecorp/kone-api-examples), which is available under its stated licence. An AI-native open-source tool in this domain would be free to implement all functional capabilities surveyed here, provided it does not replicate proprietary platform-specific API call sequences under restrictive API terms of service.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Elevator unit register with per-shaft service history, component logs, and inspection records
- Compliance calendar with jurisdiction-aware deadline tracking (ASME A17.1 North America; EN 81 Europe)
- Work order management: creation, assignment, mobile field completion, and photo attachment
- Preventive maintenance scheduling by calendar and usage triggers
- AMC contract tracking with auto-renewal reminders

**Should-have (v1.1)**
- IoT sensor data ingestion with threshold-based anomaly alerting (OEM-agnostic via BACnet/Modbus/OPC-UA)
- AI-powered predictive maintenance scoring from sensor telemetry streams
- Emergency phone number registry mapped to individual elevator units
- AHJ-facing read-only compliance portal for regulatory oversight
- Automated permit filing workflow for supported jurisdictions

**Nice-to-have (backlog)**
- Blockchain-verified or cryptographically signed inspection certificate store
- Regulatory change monitoring with AI-generated fleet impact assessment
- NLP-assisted incident report classification
- Robot/autonomous-system elevator call API
- International code support: ISO 8100 harmonisation and multi-jurisdiction management
