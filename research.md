# Elevator & Vertical Transport Management

> Candidate #242 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| LiftNet | Cloud-based vertical transportation management: work orders, technician scheduling, compliance tracking for elevators, escalators, and walkways | Commercial SaaS | Custom | Purpose-built for VT industry; broad equipment type coverage; niche market awareness |
| iFactory Elevator Module | Digitises ASME A17.1 compliance lifecycle, automated permit filing, blockchain-verified inspection records | Commercial SaaS | Custom | Strong regulatory automation; AI deadline alerts; limited to North American compliance framework |
| BRYCER (The Compliance Engine) | Fire and life safety compliance tracking including elevator inspection records and deficiency management | Commercial SaaS | Custom (AHJ-focused) | Used by Authorities Having Jurisdiction; compliance-centric; not a full maintenance platform |
| Fieldy | Field service management adapted for elevator contractors: AMC reminders, inspection scheduling, mobile checklists | Commercial SaaS | Custom | Affordable and scalable; less deep on regulatory specifics |
| SafetyCulture (iAuditor) | Digital checklists and inspection reporting for elevator safety audits | Commercial SaaS | Free–$24/seat/month | Highly flexible; generic, not elevator-specific |
| Simpro | Field service ERP used by elevator contractors for quoting, scheduling, and job management | Commercial SaaS | Custom | Strong job costing and project management; not VT-specialist |
| UpKeep | Mobile CMMS used by facilities teams to manage elevator PM schedules and work orders | Commercial SaaS | From $45/user/month | Easy to deploy; lacks elevator-specific compliance workflows |
| Otis ONE / KONE 24/7 Connected Services | OEM remote monitoring platforms providing real-time diagnostics and predictive maintenance for manufacturer's own equipment | Commercial SaaS (OEM) | Bundled with equipment contracts | Deep equipment telemetry; proprietary and locked to OEM brands |

## Relevant Industry Standards or Protocols

- **ASME A17.1 / CSA B44** — Safety Code for Elevators and Escalators; the primary North American standard governing design, installation, inspection, testing, and maintenance
- **ASME A17.1-2025** — Latest revision (2025) introducing updated safety requirements; compliance tracking software must accommodate code version management
- **EN 81 series (EU)** — European safety standard for elevators; EN 81-20 and EN 81-50 cover design and testing; relevant for international deployments
- **ADA (Americans with Disabilities Act)** — Accessibility requirements for elevator cab dimensions, controls, and signage; tracked alongside safety compliance
- **NFPA 72 / 101** — Life Safety Code; elevator recall and firefighter service requirements integrated with fire alarm systems
- **ISO 9386** — International standard for powered lifting platforms for persons with impaired mobility

## Available Research Materials

1. getFieldy (2026). *Top 5 Best Elevator Management Software in FSM Business 2026*. getfieldy.com. https://getfieldy.com/blogs/best-elevator-management-software
2. LiftNet (2026). *Vertical Transportation Management Solution*. liftnet.com. https://liftnet.com/
3. iFactory (2026). *Elevator Compliance Requirements in North America*. ifactoryapp.com. https://ifactoryapp.com/industries/hvac/elevator-compliance-requirements-north-america
4. SafetyCulture (2026). *Top 7 Elevator Service Software of 2026*. safetyculture.com. https://safetyculture.com/apps/elevator-service-software
5. BRYCER / The Compliance Engine (2026). *Elevator Inspection and Compliance Software*. thecomplianceengine.com. https://www.thecomplianceengine.com/elevator
6. ANSI Blog (2025). *ASME A17.1-2025: Safety Code for Elevators and Escalators*. blog.ansi.org. https://blog.ansi.org/ansi/asme-a17-1-2025-safety-code-elevator-csa-b44/
7. ZipDo (2026). *Top 10 Best Elevator Service Software of 2026*. zipdo.co. https://zipdo.co/best/elevator-service-software/
8. LiftNet (2026). *Understanding Elevator Code Compliance: What You Need to Know*. liftnet.com. https://liftnet.com/understanding-elevator-code-compliance-what-you-need-to-know/

## Market Research

**Market Size:** The global elevator and escalator market exceeds USD 85 billion; the specialist management and compliance software segment is a small but growing sub-segment, driven by aging infrastructure and increasing regulatory scrutiny worldwide.

**Funding:** Most specialist VT software vendors (LiftNet, iFactory) are bootstrapped or early-stage; OEM platforms (Otis ONE, KONE 24/7) are funded internally by multi-billion-dollar manufacturers. No major venture-backed pure-play has emerged yet.

**Pricing Landscape:** Pricing is largely custom and opaque; generic field service platforms (UpKeep, SafetyCulture) start at $24–$45/user/month while specialist VT platforms charge per-unit or per-portfolio models. OEM remote monitoring is bundled with maintenance contracts.

**Key Buyer Personas:** Elevator service contractors managing multi-building portfolios; building owners and facility managers responsible for compliance; Authorities Having Jurisdiction (AHJs) performing regulatory oversight; real estate investment trusts with large commercial portfolios.

**Notable Trends:** IoT-enabled remote diagnostics from OEMs are creating data lock-in; regulators are pushing for digital inspection records replacing paper logs; predictive maintenance adoption is growing to reduce costly emergency callbacks; blockchain-verified compliance records are emerging for high-stakes audits.

## AI-Native Opportunity

- AI-driven anomaly detection on elevator motor current, door operation timing, and vibration signatures to predict component failures (bearings, door operators, brake pads) weeks before breakdown
- Automated cross-jurisdiction compliance calendar that maps each unit's inspection, testing, and permit renewal deadlines against the specific code version applicable at that location
- Natural language incident reporting where technicians dictate repair notes and the system auto-classifies defect type, severity, affected component, and triggers corrective action workflows
- Portfolio-level risk scoring across hundreds of buildings, surfacing the highest-priority units for proactive maintenance investment based on age, usage cycles, and deficiency history
- Regulatory change monitoring that alerts operators when ASME A17.1 or state-specific amendments are published and flags which units in the fleet require remedial action
