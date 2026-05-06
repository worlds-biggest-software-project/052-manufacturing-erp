# Manufacturing ERP

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native manufacturing ERP combining MRP, shop floor control, IoT data ingestion, and dynamic scheduling in a single production-ready package.

Manufacturing ERP is a candidate project to build a discrete-manufacturing-focused ERP for SMB and mid-market manufacturers who are priced out of Epicor, Infor, and SAP, and underserved by general-purpose open-source ERPs. It targets plant managers, quality managers, and CFOs at $10M–$100M manufacturers who need real BOM management, MRP, shop floor execution, job costing, and ISO 9001-compliant quality workflows — without a $500K implementation.

---

## Why Manufacturing ERP?

- **Commercial manufacturing ERPs are cost-prohibitive for SMBs.** Epicor Kinetic runs $80K–$200K/year, Infor CloudSuite $100K–$300K/year, and SAP S/4HANA $500K–$5M+ to implement. Manufacturers under 50 employees are effectively excluded from the best tooling.
- **The leading open-source option lacks shop floor and AI capability.** ERPNext (MIT) provides solid BOM, work orders, and quality inspection, but has no native OPC-UA or MTConnect IoT integration and no AI scheduling, demand forecasting, or predictive quality.
- **Static MRP plans cost throughput.** Existing scheduling tools assume infinite capacity and static lead times; planners manually patch around machine breakdowns and material delays. The "plan vs. reality" gap costs an estimated 15–25% in throughput.
- **Shop floor workers are not ERP users.** Job card completions and scrap reporting still go through kiosks or paper. Adoption and data quality suffer.
- **No fully open-source manufacturing ERP combines MRP, shop floor control, IoT data ingestion, and AI-driven scheduling** in a production-ready package — a clear differentiated opportunity.

---

## Key Features

### BOM, MRP, and Production Planning

- Multi-level Bill of Materials with revision control, effectivity dates, and engineering change tracking
- Net-change and regenerative MRP with time-phased requirements and exception alerts
- Planned order generation for purchase and production orders
- Subcontracting workflow for outsourced manufacturing operations
- Capacity planning with workstation workload visualisation and bottleneck detection

### Shop Floor Execution

- Work order management: create, release, track, and close production jobs with material and labour capture
- Job card reporting for production completions, scrap, and time
- Tablet-optimised shop floor interface designed for operators with no ERP expertise
- Lot and serial traceability through the full production chain
- Backflushing of raw material consumption on work order completion

### Quality Management

- Configurable quality inspection templates that gate stock movement until approved
- Non-conformance tracking and corrective action request (CAR) workflow
- ISO 9001-compliant audit trail; design path for AS9100 (aerospace) and IATF 16949 (automotive) extensions
- First-article inspection records linked to work orders and lots

### Costing and Financials

- Job costing with actual vs. standard cost variance reporting
- Work-in-progress (WIP) valuation
- Quoting and estimating cost roll-up from BOM and routing for contract and job-shop manufacturers

### AI and IoT Integration

- AI-driven dynamic scheduling that re-optimises the production sequence in real time as capacity, tooling, and material conditions change, with explainable rescheduling decisions
- Natural-language shop floor interface (voice or chat) for operators to report job completions, scrap, and material issues; the system handles the underlying ERP transactions
- OPC-UA and MTConnect machine data ingestion for real-time shop floor visibility
- BOM construction assistant that parses CAD drawings, PDF datasheets, and part specifications to generate draft BOMs with confidence scores
- Predictive quality agent that ingests live machine parameters alongside historical inspection data to predict failures before they occur and trigger automatic work order holds

---

## AI-Native Advantage

AI is integral, not bolted on. A dynamic scheduling agent continuously re-optimises the production sequence using live capacity, skills, tooling, and material data — closing the plan-vs-reality gap that costs incumbents 15–25% throughput. A natural-language operator interface replaces kiosks and paper, dramatically improving shop floor data quality. A BOM construction assistant accelerates new product introduction by parsing engineering artifacts, and a predictive quality agent shifts QC from after-the-fact detection to prevention by correlating OPC-UA/MTConnect telemetry with inspection history. None of these capabilities ship today in any open-source manufacturing ERP, and only fragments exist (Infor Coleman, SAP AI Services) in commercial systems.

---

## Tech Stack & Deployment

The project targets self-hosted and cloud deployment to give manufacturers control over sovereignty and connectivity. Integration with shop floor equipment is built around open, royalty-free standards: **MTConnect** for CNC machine tool data and **OPC-UA** for PLC/SCADA communication. The design follows **ISA-95** for the boundary between enterprise (ERP) and plant-floor (MES) execution. Product identification and traceability use **GS1 / GTIN / UDI**. A REST API surface is expected for enterprise integration, with EDI connectors anticipated for OEM customer document exchange. Quality workflows align with **ISO 9001**, with extension paths for **AS9100** (aerospace) and **IATF 16949** (automotive). Asset management aligns with **ISO 55000**.

---

## Market Context

The global discrete manufacturing ERP market is approximately **$7.13 billion in 2026** (Business Research Insights), with the batch/process sub-market at ~$2.05 billion. The overall ERP software market is **$81.3 billion**, with manufacturing the largest vertical at ~47% of adopters and growing at a CAGR of 9–11% (Fortune Business Insights). Pricing tiers run from open source (ERPNext, OFBiz: $0–$10K) through SMB cloud ($10K–$50K), mid-market ($50K–$200K), upper mid-market ($80K–$300K+), to enterprise SAP/Oracle deployments at $500K–$5M+. Primary buyers are plant managers and VPs of Operations at $10M–$100M discrete manufacturers, quality managers requiring ISO 9001 / IATF 16949 traceability, and CFOs at contract manufacturers needing job costing and WIP valuation.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

Candidate #052 · Complexity 9/10 · Demand: High · Domain availability: Low · Category: Manufacturing ERP

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
