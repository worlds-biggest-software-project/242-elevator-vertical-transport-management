# Elevator & Vertical Transport Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, OEM-agnostic platform for inspection scheduling, compliance tracking, and incident management across elevator, escalator, and moving-walkway fleets.

Elevator & Vertical Transport Management is a candidate open-source platform for elevator service contractors, building owners, and Authorities Having Jurisdiction (AHJs) who need a unified system for vertical-transport maintenance, regulatory compliance, and predictive diagnostics. It addresses the gap between brand-locked OEM monitoring platforms (Otis ONE, KONE 24/7) and generic field-service tools that lack elevator-specific compliance workflows.

---

## Why Elevator & Vertical Transport Management?

- OEM remote-monitoring platforms (Otis ONE, KONE 24/7) deliver deep telemetry but are locked to a single manufacturer's equipment, leaving mixed-fleet portfolios without a unified view.
- Specialist VT software (LiftNet, iFactory, FIELDBOSS) is proprietary, custom-priced, and predominantly North-American — international code support (EN 81, ISO 8100) is sparse.
- Generic field-service and CMMS tools (ServiceTitan, UpKeep, SafetyCulture) cannot model an elevator shaft as an independent compliance entity with its own inspection records, code-version history, and permit lifecycle.
- AHJ-facing compliance platforms (BRYCER) are oriented to regulators rather than contractors, and elevator coverage is secondary to fire and life-safety use cases.
- No platform actively monitors ASME A17.1 and EN 81 amendments and flags which units in a fleet require remedial action when codes change.

---

## Key Features

### Unit Register & Service History

- Per-shaft asset model with independent component logs, inspection records, and service history
- Emergency phone number registry mapped to individual elevator units (an ASME A17.1 requirement)
- Multi-OEM support across elevators, escalators, and moving walkways
- AMC (Annual Maintenance Contract) tracking with auto-renewal reminders

### Compliance & Regulatory Workflow

- Jurisdiction-aware compliance calendar tied to specific ASME A17.1 and EN 81 code versions
- Automated permit-filing workflow for supported jurisdictions
- AHJ-facing read-only compliance portal for regulatory oversight without requiring AHJ tooling adoption
- Cryptographically signed or blockchain-verified inspection certificate store (backlog)
- Regulatory change monitoring with AI-generated fleet impact assessment

### Work Order & Field Service

- Work order creation, assignment, and mobile field completion with photo attachment
- Preventive maintenance scheduling by calendar and usage/meter triggers
- Offline-capable mobile app for technicians in low-connectivity buildings
- QR-code-driven fault reporting for building occupants

### IoT Telemetry & Predictive Maintenance

- OEM-agnostic IoT sensor ingestion via BACnet, Modbus, and OPC-UA
- Threshold-based anomaly alerting from motor current, door timing, vibration, and brake deceleration streams
- AI-powered predictive maintenance scoring derived from sensor telemetry
- Portfolio-level risk scoring across age, usage cycles, deficiency history, and code compliance status

### Incident Reporting & Analytics

- NLP-assisted incident report classification (defect type, component, severity, corrective action)
- Customer notification flows for scheduled visits and job completion
- Reporting on MTTR, PM completion, downtime, and AMC coverage

---

## AI-Native Advantage

AI is applied where incumbents leave gaps: anomaly detection on raw sensor streams to predict component failures weeks ahead of breakdown; an AI deadline engine that cross-references each unit's jurisdiction, equipment type, and code version to generate individualised compliance calendars; NLP-based parsing of dictated technician notes to auto-classify defects and trigger corrective actions; and automated regulatory change monitoring that flags impacted units when ASME A17.1 or state-specific amendments are published.

---

## Tech Stack & Deployment

The platform is intended to be deployable in self-hosted or cloud configurations, exposing open APIs for integration with BMS, BAS, and SCADA systems. OEM-agnostic IoT integration uses BACnet, Modbus, and OPC-UA. Where OEM telemetry is available (Otis Building Management API, KONE Elevator WebSocket / Site Monitoring APIs), the platform consumes it as one data source among many rather than as the primary integration. Standards in scope include ASME A17.1 / CSA B44, ASME A17.1-2025, EN 81-20 / EN 81-50, ADA, NFPA 72 / 101, and ISO 9386.

---

## Market Context

The global elevator and escalator market exceeds USD 85 billion; the management and compliance software sub-segment is small but growing as aging infrastructure and regulatory scrutiny increase. Pricing among incumbents is largely opaque and custom: generic field-service platforms start at USD 24–45/user/month (SafetyCulture, UpKeep), while specialist VT platforms use per-unit or per-portfolio models. Primary buyers are elevator service contractors with multi-building portfolios, building owners and facility managers, AHJs, and REITs with large commercial portfolios.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
