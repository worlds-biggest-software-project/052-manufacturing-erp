# Manufacturing ERP — Feature & Functionality Survey

> Candidate #52 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Epicor Kinetic | Commercial SaaS / On-prem | Proprietary; from $80/user/month; typical $100K–$500K/year | https://www.epicor.com |
| Infor CloudSuite Industrial (SyteLine) | Commercial SaaS | Proprietary; $100K–$300K/year | https://www.infor.com |
| SYSPRO | Commercial | Proprietary; $199/user/month subscription or perpetual | https://www.syspro.com |
| SAP S/4HANA Manufacturing | Commercial | Proprietary; $500K–$5M+ implementation | https://www.sap.com |
| Global Shop Solutions | Commercial | Proprietary; $50K–$150K implementation | https://www.globalshopsolutions.com |
| ERPNext (Frappe) | Open Source | MIT licence; self-hosted free; hosted from $50/month unlimited users | https://erpnext.com |
| Odoo Manufacturing | Open Core | Community GPL v3; Enterprise proprietary (~€20–€30/user/month) | https://www.odoo.com/app/manufacturing |
| MIE Trak Pro | Commercial | Proprietary; custom SMB pricing | https://www.mietrak.com |
| Kladana | Commercial SaaS | Proprietary; from ~$49/month | https://kladana.com |
| Apache OFBiz | Open Source | Apache Licence 2.0; free; high implementation cost | https://ofbiz.apache.org |

## Feature Analysis by Solution

### Epicor Kinetic

**Core features**
- Multi-level BOM management with revision control, effectivity dates, engineering change management, and phantom/sub-assembly support
- Net-change and regenerative MRP with multi-level pegging, time-phased requirements, and exception-based planning for purchase and production orders
- Finite and infinite capacity scheduling with Gantt chart visualisation, bottleneck analysis, and what-if simulation
- Advanced Planning and Scheduling (APS): Kinetic APS adds multiple-constraint scheduling, visual drag-and-drop scheduling, capability-based scheduling, and real-time capable-to-promise
- Shop floor data collection: job card completion, scrap reporting, and labour time capture via kiosk or mobile
- Quality management: non-conformance tracking, corrective action requests, and inspection records
- Application Studio: no-code screen customisation enabling manufacturers to tailor the UI without developer involvement
- Job costing: actual vs. standard cost variance reporting with WIP valuation

**Differentiating features**
- APS finite scheduling with real-world constraint modelling (machine capability, tooling, operator skills) is the deepest scheduling capability among mid-market manufacturing ERPs
- Application Studio provides meaningful customisation without code — critical for manufacturers with unique workflows
- Designed exclusively for discrete manufacturers: job shop, make-to-order, engineer-to-order, and mixed-mode environments are all first-class use cases
- Real-time capable-to-promise gives sales teams an accurate delivery date based on actual capacity at time of order entry

**UX patterns**
- Browser-based responsive UI replacing the legacy Epicor 10 thick client
- Role-based dashboards for plant manager, scheduler, quality manager, and CFO personas
- Shop floor kiosk mode for operator job card reporting

**Integration points**
- REST and OData APIs for enterprise system integration
- EDI connectors for OEM customer purchase orders and shipping notifications
- Microsoft 365 and Teams integration for notifications and approvals
- Epicor Connected Process Control for IoT sensor and machine data integration (OPC-UA capable)

**Known gaps**
- No native OPC-UA or MTConnect shop floor IoT data collection in standard configuration — requires Epicor Connected Process Control add-on
- AI scheduling and predictive quality are roadmap items, not shipping features as of 2026
- Implementation complexity: $100K–$500K/year cost plus $100K–$500K implementation excludes most SMBs under 50 employees
- Less depth in process manufacturing (batch, continuous) than discrete manufacturing

**Licence / IP notes**
- Fully proprietary; Epicor is PE-owned (CD&R buyout 2023)

---

### Infor CloudSuite Industrial (SyteLine)

**Core features**
- Full discrete and process manufacturing ERP: BOM, routings, work orders, MRP/MPS, production scheduling, and shop floor execution
- Pre-built vertical templates: aerospace (AS9100), automotive (IATF 16949), and industrial equipment configurations accelerate deployment
- Advanced planning with constraint-based scheduling
- Quality management with full lot/serial traceability for regulated industries
- Infor OS integration platform for cross-system connectivity
- Infor Coleman AI: embedded AI/ML features for demand forecasting and anomaly detection

**Differentiating features**
- Pre-built vertical templates for aerospace and automotive compliance reduce configuration time for regulated tier-1/tier-2 suppliers
- Infor Coleman AI provides embedded ML-based demand forecasting within the ERP — more native than Epicor's AI positioning
- Koch Industries ownership (since 2020) provides deep investment in AI and cloud infrastructure modernisation

**UX patterns**
- Infor Ming.le social collaboration layer overlaid on the ERP UI provides notifications and workflows in a modern web interface
- Role-based homepages with configurable KPI widgets

**Integration points**
- Infor OS: pre-built connectors to Salesforce, AWS, Azure, and third-party logistics
- EDI B2B platform for OEM document exchange
- OPC-UA machine connectivity via Infor Factory Track for shop floor IoT data

**Known gaps**
- Implementation complexity and cost ($100K–$300K/year) similar to Epicor; not SMB-accessible
- AI features are present but less mature than SAP's AI roadmap
- Global implementations require Infor professional services; partner ecosystem smaller than SAP or Oracle

**Licence / IP notes**
- Fully proprietary; Koch Industries subsidiary

---

### SYSPRO

**Core features**
- Manufacturing Operations Management (MOM) module: shop floor scheduling, work orders, and production tracking
- MRP with capacity planning and exception alerts
- Lot and serial number traceability throughout the supply chain
- Quality management: inspection checklists, non-conformance, and corrective action
- CMMS (Computerised Maintenance Management System): equipment maintenance scheduling integrated with production planning
- Available as subscription (SaaS) or perpetual on-prem licence

**Differentiating features**
- Integrated CMMS within the ERP is uncommon at SYSPRO's price point — most competitors require a separate maintenance management system
- Perpetual on-prem licence provides a true alternative to SaaS subscription for manufacturers with sovereignty or connectivity concerns

**UX patterns**
- Traditional ERP UI with configurable role-based entry points (SYSPRO Avanti web UI modernises the desktop experience)
- Mobile app for shop floor data capture

**Integration points**
- SYSPRO API for external system integration
- EDI connector for trading partner document exchange
- Microsoft Power BI integration for reporting

**Known gaps**
- Less powerful APS than Epicor Kinetic for complex constraint-based scheduling
- AI features minimal compared to Infor Coleman or SAP AI roadmap
- Smaller global partner ecosystem than SAP, Oracle, or Epicor

**Licence / IP notes**
- Fully proprietary; subscription and perpetual licence options

---

### SAP S/4HANA Manufacturing

**Core features**
- Production Planning and Detailed Scheduling (PP/DS): constraint-based scheduling with deep integration to HANA in-memory database
- Quality Management (QM): ISO 9001, AS9100, IATF 16949-compliant inspection management with full lot traceability
- Plant Maintenance (PM): integrated asset management and predictive maintenance
- Digital Manufacturing Cloud (DMC): MES-level shop floor execution integrated with S/4HANA
- SAP AI Business Services: embedded ML for predictive quality, demand forecasting, and anomaly detection

**Differentiating features**
- Only ERP with MES-level shop floor execution (DMC) and ERP planning in a single vendor solution — eliminates ISA-95 integration complexity
- HANA in-memory database enables real-time MRP runs that process in minutes rather than overnight batch windows
- Broadest compliance coverage: every major quality standard (ISO 9001, AS9100, IATF 16949, GxP) is supported natively

**UX patterns**
- SAP Fiori launchpad: role-based tile interface providing a modern web experience on HANA
- Steep learning curve; requires extensive training even with Fiori modernisation

**Integration points**
- SAP Integration Suite: pre-built connectors to virtually every enterprise system
- OPC-UA and MTConnect shop floor connectivity via SAP Asset Intelligence Network
- Cloud hyperscaler integrations (AWS, Azure, GCP)

**Known gaps**
- Overkill and cost-prohibitive for manufacturers below $100M revenue; $500K–$5M+ implementation cost
- SAP ecosystem lock-in is the most significant of any tool surveyed
- Implementation requires SAP-certified consultants; global shortage of S/4HANA expertise inflates project costs

**Licence / IP notes**
- Fully proprietary; enterprise licence and cloud subscription options

---

### ERPNext (Frappe)

**Core features**
- MIT-licensed open-source manufacturing module: multi-level BOM with revision history and BOM Comparison Tool, work orders with automatic material and job card generation, job card completion reporting with real-time workstation status, and quality inspection templates that gate stock movement until inspection is approved
- Production planning: material requirements analysis from sales orders and production schedules
- Capacity planning: workstation workload tracking for scheduling optimisation
- Backflushing: automatic raw material consumption recording on work order completion
- Subcontracting: purchase order workflow for outsourced manufacturing operations
- Frappe framework: DocType-based customisation enabling new fields, forms, and workflows without code

**Differentiating features**
- Only MIT-licensed manufacturing ERP with meaningful depth — BOM, work orders, job cards, quality inspection, and subcontracting are all production-ready
- Per-site (not per-user) Frappe Cloud pricing makes TCO radically lower than any commercial alternative at team scale
- BOM Comparison Tool is a practical feature that commercial ERPs often charge for as an add-on

**UX patterns**
- DocType-based forms are functional but less polished than commercial competitors
- Web-based and mobile-accessible; Frappe apps are progressive web apps
- Shop floor interface is form-based; no dedicated kiosk mode comparable to Epicor

**Integration points**
- Comprehensive REST API covering all DocTypes
- MTConnect integration via community modules (not native — requires custom implementation)
- Payment, bank, and eCommerce connectors via Frappe Marketplace

**Known gaps**
- No native OPC-UA or MTConnect IoT integration — machine data collection requires custom development
- No AI features: no ML-based demand forecasting, predictive quality, or AI scheduling as of 2026
- APS finite scheduling is limited — capacity planning exists but constraint-based optimisation is manual
- UX polish and enterprise governance features require significant configuration investment

**Licence / IP notes**
- ERPNext: MIT licence — most permissive open-source licence; no copyleft, commercial use unrestricted, redistribution unrestricted
- Frappe framework: MIT licence
- Frappe Cloud hosted service: commercial pricing

---

### Odoo Manufacturing

**Core features**
- BOM management with components, operations, and by-products
- Work centres and routing configuration for production floor layout
- Work order execution with real-time progress tracking
- Quality control points configurable at BOM operation level
- Preventive maintenance integration with production planning
- Product Lifecycle Management (PLM) module for engineering change management
- Integrated with Odoo's full suite: purchasing, inventory, accounting, and sales on the same platform

**Differentiating features**
- Tightest integration between manufacturing and commercial operations of any tool surveyed — a confirmed sales order can automatically trigger a production work order with zero manual steps
- PLM module for engineering change management is an add-on most SMB ERPs lack
- Odoo 20 agentic AI roadmap includes proactive inventory replenishment and production trigger automation

**UX patterns**
- Consistent Odoo design language across manufacturing and commercial modules — lower cross-module training burden
- Kanban-style work order board for production floor overview
- Tablet-optimised work order execution screen for operator use

**Integration points**
- Odoo REST API
- IoT Box: Odoo-provided hardware for connecting industrial machines to work orders via digital I/O signals (limited to simple trigger events, not full OPC-UA data streams)
- Barcode scanner integration for material movements and job card completion

**Known gaps**
- Manufacturing depth below dedicated manufacturing ERPs (Epicor, Infor) for complex job shop and engineer-to-order scenarios
- APS and constraint-based scheduling are not available in Odoo — only basic scheduling by work centre
- OPC-UA and MTConnect integration requires third-party development; IoT Box is limited
- Community edition (GPL v3) excludes PLM, maintenance, and quality modules — these are Enterprise-only

**Licence / IP notes**
- Community: GPL v3 (core and community modules — open source)
- Enterprise manufacturing modules (PLM, quality, maintenance): proprietary Odoo Enterprise licence
- IoT Box: open-source Raspberry Pi firmware

---

### Global Shop Solutions

**Core features**
- Quoting and estimating with cost roll-up from BOM and routing
- Job costing with actual vs. estimate variance tracking
- Shop floor data collection via barcode kiosk
- Production scheduling with visual Gantt board
- Quality management and first-article inspection
- Purchasing and inventory integrated with production planning

**Differentiating features**
- Purpose-built for US job shops and custom manufacturers (make-to-order, engineer-to-order)
- Quoting and estimating from BOM/routing is core to the product, not an add-on — critical for contract manufacturers pricing custom jobs
- Strong shop floor data collection tuned to job shop workflows

**UX patterns**
- Traditional ERP interface; not the most modern but deeply understood by US job shop users
- Barcode kiosk optimised for shop floor operator use without ERP knowledge

**Integration points**
- EDI for OEM customer orders
- CAD/CAM data import for BOM and routing creation

**Known gaps**
- US-focused; limited international compliance and localisation
- No AI features
- Less powerful MRP and capacity planning than Epicor or Infor for complex multi-plant scenarios

**Licence / IP notes**
- Fully proprietary; no open-source components

---

### Apache OFBiz

**Core features**
- Open-source ERP framework with manufacturing capabilities: MRP, BOM, work orders, and inventory
- Highly modular architecture enabling selective deployment of components
- Active Apache Software Foundation community maintenance

**Differentiating features**
- Apache Licence 2.0: most permissive major open-source licence — no copyleft, suitable for proprietary commercial products built on OFBiz
- Long track record: OFBiz has been maintained since 2001 and is a proven foundation for custom ERP builds

**UX patterns**
- Dated Groovy/FTL-based UI; significant developer work required to create a modern interface
- Not SMB-accessible out of the box — requires extensive developer customisation before deployment

**Integration points**
- Service-oriented architecture enables integration via web services
- Flexible data model supports custom integration patterns

**Known gaps**
- Not SMB-friendly without significant developer investment — implementation cost often exceeds commercial alternatives
- Community activity level lower than ERPNext or Odoo Community
- No AI features; roadmap shows no AI investment

**Licence / IP notes**
- Apache Licence 2.0: permissive, allows proprietary use without copyleft; attribution required

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-level BOM management with revision control and engineering change management
- MRP: net-change and regenerative material requirements planning with time-phased requirements and exception alerting
- Work order management: create, release, track, and close production jobs with material and labour capture
- Job card reporting: operator-facing interface for recording production completions, scrap, and time
- Shop floor data collection: barcode or kiosk-based capture for stock movements and job progress
- Job costing: actual vs. standard/estimate cost variance reporting with WIP valuation
- Quality inspection with configurable inspection templates and non-conformance tracking
- ISO 9001-compliant audit trail for lot and serial traceability

### Differentiating Features
- Constraint-based APS with finite capacity scheduling and what-if simulation (Epicor Kinetic, SAP PP/DS)
- MES-level shop floor execution integrated with ERP planning in a single system (SAP DMC)
- Integrated CMMS/asset management within the manufacturing ERP (SYSPRO, SAP PM)
- Native OPC-UA and MTConnect machine data connectivity (SAP, Infor — commercial; absent from all OSS options)
- Real-time capable-to-promise for order delivery date calculation (Epicor APS)
- AI/ML-based demand forecasting and predictive quality embedded in the ERP (Infor Coleman, SAP AI Services)
- No-code screen customisation for manufacturer-specific workflows (Epicor Application Studio, Odoo Studio)

### Underserved Areas / Opportunities
- AI-driven dynamic scheduling: all scheduling tools generate plans that assume static conditions; none continuously re-optimise the production sequence as machine breakdowns, absent operators, and material delays occur in real time
- Natural-language shop floor interface: operators are not ERP users; AI voice or chat input for job card completion, scrap reporting, and material issues would improve shop floor data quality and system adoption dramatically
- Automated BOM construction from engineering artifacts: creating BOMs from CAD drawings and PDF datasheets is still entirely manual; AI parsing of engineering documents to generate draft BOMs would accelerate product introduction
- Predictive quality integrated with machine data: QC modules record defects after the fact; AI ingesting real-time OPC-UA machine parameters alongside inspection results could predict quality failures before they occur
- Open-source gap: no fully open-source manufacturing ERP combines MRP, shop floor control, IoT data ingestion, and AI-driven scheduling — ERPNext is the closest but lacks native IoT connectivity and AI scheduling

### AI-Augmentation Candidates
- Dynamic scheduling agent: continuously monitor live capacity (machine availability, operator skills, tooling status) and material availability to re-optimise the production sequence in real time, explaining each rescheduling decision
- BOM construction assistant: parse CAD drawings, PDF datasheets, and part specifications to auto-generate draft BOM entries with confidence scores, flagging discrepancies with purchasing and inventory data for engineer review
- Predictive quality agent: ingest real-time OPC-UA/MTConnect machine parameters alongside historical inspection data to predict quality failures before they occur and automatically trigger work order holds
- Natural-language shop floor interface: accept voice or chat input from operators reporting job completions, scrap events, and material issues — handling the ERP transaction automatically

## Legal & IP Summary

Two open-source manufacturing ERP options exist: ERPNext (MIT licence — most permissive; unrestricted commercial use and redistribution) and Apache OFBiz (Apache Licence 2.0 — permissive; attribution required; no copyleft). Odoo Community manufacturing modules are GPL v3/LGPL v3 (copyleft — modifications must be released under the same licence); Odoo Enterprise manufacturing modules (PLM, quality, maintenance) are proprietary. All other surveyed tools are fully proprietary commercial products. ISA-95 is an open ANSI/ISA standard with no IP restrictions. MTConnect is royalty-free and openly available. OPC-UA is an open standard published by the OPC Foundation. ISO 9001, AS9100, and IATF 16949 are ISO/IATF standards available for purchase; compliance itself is not IP-restricted. Manufacturing ERP systems process personally identifiable information for employees (time records, payroll-adjacent data) which is subject to GDPR and jurisdiction-specific labour data regulations.

## Recommended Feature Scope

**Must-have (MVP)**:
- Multi-level BOM with revision control and engineering change tracking
- MRP with time-phased requirements, exception alerts, and planned order generation
- Work order management with material issue, operation tracking, and job card completion
- Shop floor data collection interface (tablet-optimised for operator use; no ERP expertise required)
- Job costing with actual vs. standard cost variance and WIP valuation
- Quality inspection with configurable templates and non-conformance tracking
- Lot and serial traceability through the full production chain

**Should-have (v1.1)**:
- AI-driven dynamic scheduling: re-optimise production sequence in real time as capacity and material conditions change
- Natural-language shop floor interface: operators report production events in plain language; system handles ERP transactions
- OPC-UA / MTConnect machine data integration for real-time shop floor visibility
- Subcontracting workflow for outsourced manufacturing operations
- Capacity planning with workstation workload visualisation and bottleneck detection

**Nice-to-have (backlog)**:
- BOM construction assistant: AI parsing of engineering documents to generate draft BOMs with confidence scores
- Predictive quality agent: ingest machine parameters and inspection history to predict quality failures before they occur
- Integrated CMMS: preventive and predictive maintenance scheduling linked to production planning
- PLM integration for engineering change management with BOM version control
- Advanced APS with finite constraint scheduling and what-if simulation for complex multi-machine job shops
