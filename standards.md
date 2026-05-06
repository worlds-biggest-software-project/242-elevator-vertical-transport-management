# Standards & API Reference

> Project: Elevator & Vertical Transport Management · Generated: 2026-05-03

## Industry Standards & Specifications

### Safety Codes — North America

**ASME A17.1 / CSA B44 — Safety Code for Elevators and Escalators**
The primary North American standard governing design, installation, testing, inspection, and maintenance of elevators, escalators, and moving walks. Compliance software must track the specific code edition adopted by each jurisdiction, as states and cities adopt new editions on different schedules. The 2025 edition (A17.1-2025) introduced updated safety requirements.
- https://www.asme.org/resources/a17-elevators-and-escalators
- https://blog.ansi.org/ansi/asme-a17-1-2025-safety-code-elevator-csa-b44/

**ASME A17.3 — Safety Code for Existing Elevators and Escalators**
Applies specifically to elevators already installed and in service. Governs the upgrade and retrofit requirements when older units are brought into compliance with current safety requirements. Compliance platforms must distinguish A17.1 (new installations) from A17.3 (existing equipment) requirements.
- https://blog.ansi.org/ansi/asme-a17-3-2023-code-existing-elevators-escalators/

**ASME A17.7 / CSA B44.7 — Performance-Based Safety Code for Elevators and Escalators**
A performance-based alternative to the prescriptive A17.1 code, permitting innovative designs that satisfy safety objectives through alternative means. Relevant for platforms managing modernisations and technology upgrades.
- https://www.asme.org/codes-standards/find-codes-standards?designatorCategory=A17

### Safety Codes — Europe & International

**EN 81 Series — Safety Rules for the Construction and Installation of Lifts (CEN/CENELEC)**
The European standard family governing lift safety. EN 81-20 covers requirements for design and construction of passenger and goods passenger lifts; EN 81-50 specifies rules for examination and testing. EN 81-21 addresses new lifts in existing buildings. EN 81-28 covers remote alarm on passenger and goods passenger lifts. Compliance management software for European or international deployments must track EN 81 edition and national transposition status.
- https://dazenelevator.com/elevator-safety-codes-guide/

**ISO 8100-1 / ISO 8100-2 — Lifts for the Transport of Persons and Goods**
The emerging ISO global harmonisation standard for electric lifts (Part 1) and hydraulic lifts (Part 2), developed jointly by ISO/TC 178 and CEN/TC 10, and based on EN 81-20/-50. Represents the future global unified elevator safety code, progressively superseding regional codes. International deployments should track ISO 8100 adoption status by country.
- https://elevatorworld.com/article/development-of-the-iso-prescriptive-elevator-code/

**ISO 8100-32:2020 — Lifts for Transport of Persons and Goods: Design Rules for Lifts with Reduced Pit, Reduced Headroom, or Both**
Specific guidance for space-constrained lift installations. Relevant for urban retrofit projects.
- https://elevatorworld.com/article/iso-8100-322020-guidance/

### Accessibility Standards

**ADA (Americans with Disabilities Act) — Accessible Design Standards**
US federal accessibility requirements governing elevator cab dimensions, control panel height and Braille, door timing, and audible/visual signals. Compliance platforms managing US commercial building portfolios must track ADA compliance status per unit.
- https://www.ada.gov/

**ISO 9386 — Powered Stairlifts and Inclined and Vertical Lifting Platforms for Persons with Impaired Mobility**
International standard for powered lifting platforms (not full passenger elevators) serving persons with disabilities. Relevant for platforms managing residential stairlifts, platform lifts, and accessibility equipment alongside full passenger elevators.

### Fire Safety Integration Standards

**NFPA 72 — National Fire Alarm and Signalling Code**
Governs integration of fire alarm systems with elevator controls, including Phase I and Phase II emergency recall operation. Compliance platforms must track that elevator recall interface wiring and programming meets NFPA 72 and ASME A17.1 Section 2.27 requirements.
- https://www.nfpa.org/codes-and-standards/nfpa-72

**NFPA 101 — Life Safety Code**
Governs fire egress requirements including elevator use during building emergencies. Relevant for compliance software tracking building-wide life safety certification.
- https://www.nfpa.org/codes-and-standards/nfpa-101

### Energy Performance Standards

**ISO 25745-1 / 25745-2 / 25745-3 — Energy Performance of Lifts, Escalators, and Moving Walks**
Three-part ISO standard covering energy measurement methodology, classification of energy efficiency, and energy calculation methods. Increasingly relevant as building owners face ESG reporting requirements. Compliance platforms can incorporate energy performance ratings alongside safety compliance records.
- https://www.iso.org/

### Building Automation Protocols

**ANSI/ASHRAE 135 / ISO 16484-5 — BACnet: A Data Communication Protocol for Building Automation and Control Networks**
The dominant open standard for building automation network communication. Elevator controllers expose floor call status, fault conditions, and operational state via BACnet objects. The BACnet Elevator Working Group is actively developing extensions for elevator and escalator monitoring. An open-source VT management platform should support BACnet as a primary integration protocol for connecting to elevator controllers and building management systems.
- https://bacnet.org/about-bacnet-standard/
- https://www.iso.org/standard/79630.html (ISO 16484-6:2020 conformance testing)

**IEC 61968 / IEC 61970 — Common Information Model (CIM)**
While primarily used in power systems, CIM provides a reference for open data model design when building standards-aligned asset data models for building equipment including elevators.

**OPC UA (IEC 62541) — OPC Unified Architecture**
The leading open interoperability standard for industrial IoT and building automation data exchange. OPC UA protocol converters bridge proprietary elevator controller interfaces to BMS and cloud analytics platforms. Elevator IoT retrofit solutions increasingly expose sensor data via OPC UA. Relevant for the platform's IoT data ingestion layer.
- https://opcfoundation.org/

**Modbus (Modbus Application Protocol Specification)**
Legacy serial and TCP/IP protocol widely used in elevator controllers and motor drives for status register access. Many older elevator controllers expose fault codes and operational state via Modbus TCP. The platform should support Modbus TCP as an ingestion protocol for legacy equipment.
- https://modbus.org/

### Security & Compliance Standards

**ISO/IEC 27001 — Information Security Management Systems**
Baseline information security management standard. IoT-connected elevator platforms collecting building operational data must address ISMS requirements, particularly relevant for OEM-agnostic sensor data capture.
- https://www.iso.org/isoiec-27001-information-security.html

**NIST Cybersecurity Framework (CSF 2.0)**
US federal guidance for managing cybersecurity risk in critical infrastructure. As elevators are increasingly IoT-connected and integrated with building systems, elevator management software handling remote control interfaces should align with NIST CSF identify/protect/detect/respond/recover functions.
- https://www.nist.gov/cyberframework

**OWASP API Security Top 10**
Reference for securing REST and WebSocket APIs used for elevator remote monitoring and BMS integration. Applicable to any public-facing or developer-portal API exposed by the platform.
- https://owasp.org/www-project-api-security/

**GDPR (EU Regulation 2016/679)**
Applicable where the platform processes personal data of building occupants or technicians in EU jurisdictions. Relevant for telemetry logs that could identify individuals through timing and location of elevator calls.
- https://gdpr-info.eu/

### API & Data Specification Standards

**OpenAPI Specification 3.x (OAS3)**
The de-facto standard for documenting REST APIs. All major elevator OEM developer portals (Otis, KONE, Schindler, TK Elevator) expose their APIs via documented REST interfaces. An open-source platform should provide an OAS3-compliant API specification.
- https://spec.openapis.org/oas/latest.html

**ISO 8601 — Date and Time Format**
Relevant for cross-jurisdiction compliance deadline management where inspection date formats must be unambiguous across North American and European systems.

---

## Similar Products — Developer Documentation & APIs

### KONE 24/7 Connected Services — API Platform

- **Description:** KONE's cloud-connected maintenance and monitoring platform exposing real-time elevator telemetry and call control via REST and WebSocket APIs.
- **API Documentation:** https://dev.kone.com/ and https://developer-api.kone.com/
- **SDKs / Libraries:** TypeScript example code: https://github.com/konecorp/kone-api-examples
- **Developer Guide:** https://developer.kone.com/s/news/elevator-call-api-MCFPDJMTEQIBGGZNULCIPNH47YBQ
- **Standards:** REST/JSON; WebSocket (real-time call execution and status streaming)
- **Authentication:** OAuth 2.0 via KONE API Portal account; contract with KONE required for production access

Key APIs:
- **Elevator Call API (WebSocket v2)**: Execute elevator calls; receive real-time status of call and assigned car
- **Site Monitoring API**: Fetch real-time lift status, call state, deck position, door state, and equipment information
- **Service Order API**: Integrate open/completed service orders and repair records with BMS or asset management systems

---

### Otis ONE — Building Management API

- **Description:** Otis ONE IoT monitoring platform with Building Management API exposing elevator availability, maintenance insights, and floor-level traffic data for BMS/SCADA integration.
- **API Documentation:** https://www.otis.com/en/us/building-management-api
- **Developer Portal:** https://developers.otis.com/ and https://developerstudio.otis.com/
- **BMS API Appendix:** https://www.otis.com/documents/d/otis-2/bms-api-5-10-2023-final
- **Standards:** REST/JSON; integration targets BMS, BAS, and SCADA systems
- **Authentication:** API keys via Otis Developer Portal; active Otis service contract required

Key capabilities:
- Real-time elevator availability and status data feed to BMS
- Maintenance insight data export (service history, upcoming PM triggers)
- Two-way robot/autonomous-vehicle interface (elevator-to-robot and robot-to-elevator)
- Sandbox environment available for development and testing

---

### TK Elevator — MAX Customer API

- **Description:** TK Elevator's MAX Smart Maintenance IoT platform exposing elevator and escalator operational data and work order management via customer API.
- **Developer Portal:** https://developer.ea.tkelevator.com/
- **API List:** https://developer.ams.tkelevator.com/apis
- **Getting Started:** https://developer.ea.tkelevator.com/getting-started
- **Marketing Overview:** https://www.tkelevator.com/global-en/products/digital-solutions/api/
- **Standards:** REST/JSON
- **Authentication:** API key subscription via developer portal; requires MAX Smart Maintenance package

Key APIs:
- **IoT API (Add-on)**: Elevator operational data feed for BMS integration; available as MAX package add-on
- **Work Order Management (WOM) API**: Integrates ticketing into BMS; enables building staff to notify TKE of malfunctions directly from BMS interface
- **Passenger Call API**: Allows passengers to call elevators from smartphones
- **Autonomous Robot API**: Enables robots and delivery systems to call and ride elevators autonomously

---

### Schindler — BuilT-In API / Developer Portal

- **Description:** Schindler's developer platform exposing elevator connectivity and building integration APIs as part of the Schindler Ahead IoT solution.
- **Developer Portal:** https://developer.schindler.com/
- **Package Catalogue:** https://developer.schindler.com/packages
- **BuilT-In API Overview:** https://www.jardineschindler.com/content/dam/website/jsg/docs/schindler-built-in-api.pdf
- **Standards:** REST/JSON
- **Authentication:** Account-based via Schindler Developer Portal; requires Schindler service relationship

Key capabilities:
- Elevator connectivity and real-time status data export
- Building integration for smart building platforms
- Robot connection API for autonomous delivery and cleaning systems

---

### ServiceTrade — Open API

- **Description:** Commercial field service management platform used by elevator and fire/life safety contractors, with open REST API for integration with ERP, CRM, and compliance reporting systems.
- **API Documentation:** https://servicetrade.com/products/servicetrade-platform/features/integrations/
- **Standards:** REST/JSON; OpenAPI-documented
- **Authentication:** API keys
- **Notable Integration:** BRYCER Compliance Engine integration for automated AHJ reporting from contractor field service data

---

### BRYCER (The Compliance Engine) — Integration API

- **Description:** AHJ compliance tracking platform with API integration allowing field service tools (e.g., ServiceTrade) to automatically push inspection reports and deficiency records to AHJ portals.
- **Integration Overview:** https://servicetrade.com/company-news/servicetrade-announces-integration-with-the-compliance-engine-by-brycer-to-automate-ahj-reporting/
- **Platform:** https://www.thecomplianceengine.com/
- **Standards:** REST/JSON integration with partner FSM platforms
- **Authentication:** Partner API agreement with BRYCER required

---

### UpKeep — CMMS REST API

- **Description:** Mobile-first CMMS with REST API and IoT sensor ingestion, used by facilities teams managing elevator preventive maintenance schedules.
- **API Documentation:** https://upkeep.com/ (developer documentation available to subscribers)
- **Standards:** REST/JSON; Zapier integration; IoT sensor threshold-based work order triggers
- **Authentication:** API keys

---

### SafetyCulture (iAuditor) — API & Webhooks

- **Description:** Digital inspection platform with REST API and webhooks enabling integration of elevator inspection data with CMMS, BMS, and reporting tools.
- **API Documentation:** https://developer.safetyculture.com/
- **Standards:** REST/JSON; Webhook event streams; Zapier integration
- **Authentication:** OAuth 2.0 and API keys
- **Notable Integration:** Pre-built connectors for Slack, Microsoft Teams, Power BI, Salesforce, and CMMS platforms

---

## Notes

**Emerging Standards Gap — Multi-OEM Telemetry Interoperability**: No published open standard exists for normalising elevator telemetry data across OEM brands. Each OEM (Otis, KONE, Schindler, TK Elevator) exposes a proprietary API with its own data model. An open-source platform aiming for OEM-agnostic predictive maintenance would need to define or adopt a normalised data model (potentially aligned with BACnet elevator objects or OPC UA) and implement per-OEM adapters.

**ISO 8100 Global Harmonisation**: The ISO/TC 178 and CEN/TC 10 joint development of ISO 8100 as a global prescriptive elevator code is ongoing. As this standard is adopted by additional jurisdictions, compliance management software will need to expand from regional code management (ASME A17.1, EN 81) to a single harmonised global code model with jurisdictional override layers.

**BACnet Elevator Extension**: The BACnet Elevator Working Group is actively extending the standard to cover elevator and escalator monitoring objects. Monitoring this working group's progress (bacnet.org) is recommended for teams building an open integration layer.

**Robot/Autonomous Vehicle Integration**: All four major OEMs now offer robot-facing elevator APIs. IEEE and ISO/TC 299 (robotics) are beginning to address standards for building-elevator-robot interoperability, but no finalised standard exists as of 2026. This represents an early opportunity for open protocol development.
