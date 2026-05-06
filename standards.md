# Standards & API Reference

> Project: Manufacturing ERP · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ANSI/ISA-95 / IEC 62264 — Enterprise-Control System Integration**
- Standard series: ISA-95 Parts 1–6; international equivalent IEC 62264
- Official URL: https://www.isa.org/standards-and-publications/isa-standards/isa-95-standard
- The foundational reference architecture defining the interface between enterprise ERP systems and plant-floor MES/control systems. Divides manufacturing operations into four functional domains: Production Operations Management, Quality Operations Management, Maintenance Operations Management, and Inventory Operations Management. Every manufacturing ERP that claims MES integration is expected to implement or reference the ISA-95 object models. The 2025 edition (ANSI/ISA 95.00.01-2025 / IEC 62264-1 Mod) was published in late 2025, updating the 2010 revision with modernised IT/OT convergence terminology and expanded support for cloud and digital factory architectures.

**ISO 9001:2015 — Quality Management Systems**
- Official URL: https://www.iso.org/standard/62085.html
- The globally adopted QMS standard applicable to any manufacturing organisation. Drives QC module requirements in manufacturing ERP: non-conformance (NCR) tracking, corrective action requests (CAR/CAPA), inspection checklists, internal audit records, and document control. Mandatory for any manufacturing ERP targeting suppliers that hold or seek ISO 9001 certification. ISO 9001 is scheduled for a 2026 revision; the update consolidates climate change risk considerations and aligns with the High-Level Structure (HLS) common to all ISO management system standards.

**AS9100 Rev D — Aerospace Quality Management Systems**
- Official URL: https://www.sae.org/standards/content/as9100d/
- The aerospace industry's extension of ISO 9001, adding requirements for configuration management, first-article inspection (FAI), production part approval (PPAP equivalents), risk management, and supply chain traceability. Manufacturing ERPs serving aerospace tier-1/tier-2 suppliers must support the additional AS9100 record types and traceability chains. Maintained by the International Aerospace Quality Group (IAQG) and SAE International.

**IATF 16949:2016 — Automotive Quality Management Systems**
- Official URL: https://www.iatfglobaloversight.org/iatf-169492016/
- The automotive industry's extension of ISO 9001, mandating Advanced Product Quality Planning (APQP), Failure Mode and Effects Analysis (FMEA), Production Part Approval Process (PPAP), Measurement Systems Analysis (MSA), and Statistical Process Control (SPC). Manufacturing ERPs targeting automotive tier-1/tier-2 suppliers must support these methodologies. Maintained by the International Automotive Task Force (IATF).

**ISO 55000:2024 — Asset Management**
- Official URL: https://www.iso.org/standard/83053.html
- The asset management standard governing the management of physical plant equipment and infrastructure. Relevant to the CMMS (Computerised Maintenance Management System) module of manufacturing ERP: preventive maintenance schedules, asset register, work orders for equipment repair, and lifecycle costing. The 2024 revision (updated from ISO 55000:2014) strengthens guidance on risk-based asset management decision-making. Manufacturing ERPs with integrated EAM/CMMS capability reference ISO 55000 as their maintenance management framework.

**ISO 15531 — MANDATE (Manufacturing Management Data)**
- Part 1 URL: https://www.iso.org/standard/28144.html
- An ISO TC 184/SC 4 standard defining industrial manufacturing management data including resources management data and flow management data. MANDATE provides data model standards relevant to BOM structures, routing, and production order data exchange between systems. Used in conjunction with STEP (ISO 10303) for product data representation.

---

### W3C & IETF Standards

**RFC 9110 / RFC 9112 — HTTP Semantics and HTTP/1.1**
- Official URL: https://www.rfc-editor.org/rfc/rfc9110 and https://www.rfc-editor.org/rfc/rfc9112
- The foundational HTTP specification governing all REST API communication in manufacturing ERP systems. RFC 9110 (2022) supersedes RFC 7231 and defines the semantics of HTTP methods, status codes, and headers. All ERP REST APIs (Epicor Kinetic, ERPNext, Odoo, SAP, Infor) operate over HTTP/1.1 or HTTP/2 and must comply with these semantics.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- Official URL: https://www.rfc-editor.org/rfc/rfc6749
- The OAuth 2.0 standard governing delegated API authentication. All manufacturing ERP APIs that expose REST endpoints for external system integration use OAuth 2.0 (or its successor OAuth 2.1, in draft as of 2026) for authorising third-party applications. Relevant for MES integration, IoT data ingestion connectors, and AI agent access to ERP data.

**RFC 7519 — JSON Web Token (JWT)**
- Official URL: https://www.rfc-editor.org/rfc/rfc7519
- JWT is the token format used by OAuth 2.0 bearer tokens in most modern ERP API implementations. Manufacturing ERP integrations (shop floor kiosks, IoT gateways, AI agents) commonly authenticate with JWT access tokens issued by the ERP identity provider.

**RFC 7807 — Problem Details for HTTP APIs**
- Official URL: https://www.rfc-editor.org/rfc/rfc7807
- A standard format for structured error responses from REST APIs. Relevant when designing manufacturing ERP API responses for integration clients that must handle errors programmatically (e.g., an AI scheduler reacting to a work order creation failure).

---

### Data Model & API Specifications

**OpenAPI Specification 3.1 / 3.2**
- Official URL: https://spec.openapis.org/oas/v3.1.0.html (v3.1) and https://spec.openapis.org/oas/v3.2.0.html (v3.2)
- The de-facto standard for describing REST API endpoints, request/response schemas, authentication schemes, and data models in machine-readable YAML or JSON. Epicor Kinetic publishes OpenAPI/Swagger definitions for all its REST services; SAP API Business Hub exposes OData V4 service metadata; ERPNext and Odoo provide partial OpenAPI coverage. An AI-native manufacturing ERP should publish a complete OpenAPI 3.1 spec to enable AI agents, integration platforms, and SDK code generation.

**OData V4 (ISO/IEC 20802)**
- Official URL: https://www.odata.org/ and https://docs.oasis-open.org/odata/odata/v4.01/odata-v4.01-part1-protocol.html
- Open Data Protocol, standardised by OASIS and ISO/IEC 20802. SAP S/4HANA, SYSPRO, and Microsoft Dynamics 365 all expose their manufacturing data (production orders, BOMs, work centres) via OData V4 endpoints. OData adds query capabilities (filtering, sorting, pagination, expansion of related entities) on top of standard REST that are valuable for ERP data access patterns.

**ISA-95 B2MML (Business to Manufacturing Markup Language)**
- Official URL: https://www.mesa.org/resources/b2mml/
- The XML schema implementation of the ISA-95 data models, maintained by MESA International. B2MML defines the XML structures for production schedules, work orders, material lots, personnel, and equipment data exchanged between ERP and MES systems. An AI-native manufacturing ERP targeting MES integration should support B2MML as a data exchange format.

---

### Industrial Connectivity Standards

**OPC UA — IEC 62541 (OPC Unified Architecture)**
- Official URL: https://opcfoundation.org/about/opc-technologies/opc-ua/ and https://opcfoundation.org/markets-collaboration/mtconnect/
- The international industrial machine-to-machine communication standard, standardised as IEC 62541 (multi-part series). OPC UA provides secure, reliable, platform-agnostic data exchange between PLCs, CNC machines, robots, SCADA systems, and enterprise applications including ERP. The 2026 updates include IEC 62541-9:2026 (Alarms and Conditions) and EN IEC 62541-14:2026 (PubSub Communication Model). An AI-native manufacturing ERP that ingests real-time shop floor data must implement an OPC UA client or server interface. The OPC Foundation maintains a free specification download after account registration at opcfoundation.org.

**MTConnect Standard (ANSI/MTC)**
- Official URL: https://www.mtconnect.org/ and https://docs.mtconnect.org/
- An open, royalty-free standard for CNC machine tool and manufacturing equipment data communication, published by the MTConnect Institute. MTConnect Version 2.2 (current as of 2026) uses a model-based SysML specification with XML and JSON schema outputs. MTConnect devices expose an HTTP/XML agent that ERP systems can poll for machine state, tool usage, cycle times, and production counts. A joint MTConnect–OPC UA Companion Specification (https://www.mtconnect.org/opc-ua-companion-specification) enables unified OPC UA clients to consume MTConnect device data, bridging the two standards. An AI-native manufacturing ERP should implement both MTConnect polling and the OPC UA companion specification for broadest machine compatibility.

---

### EDI & Supply Chain Standards

**ANSI X12 EDI — ASC X12**
- Official URL: https://x12.org/
- The North American EDI standard covering all structured B2B document exchange for manufacturing supply chains: X12 850 (Purchase Order), X12 856 (Advance Ship Notice), X12 810 (Invoice), X12 862 (Shipping Schedule), X12 830 (Planning Schedule). Manufacturing ERPs serving automotive OEMs (e.g., Ford, GM, Toyota) are required to support X12 EDI for receiving production schedules and shipping notifications. A manufacturing ERP targeting US job shops and contract manufacturers must provide X12 EDI connectivity or an integration with a third-party EDI translator.

**UN/EDIFACT — ISO 9735**
- Official URL: https://unece.org/trade/uncefact/introducing-unedifact
- The international EDI standard for structured B2B document exchange, used outside North America (Europe, Asia, automotive, aerospace). EDIFACT DELFOR (Delivery Forecast) and DESADV (Despatch Advice) are the equivalents of X12 830/856. Manufacturing ERPs targeting European manufacturers or global tier-1 suppliers must support EDIFACT.

**GS1 Standards — GTIN, GLN, SSCC**
- Official URL: https://www.gs1.org/standards
- GS1 provides globally unique identification for trade items (GTIN), locations (GLN), and logistics units (SSCC). Critical for lot/serial traceability in manufacturing ERP: GTIN identifies each manufactured part number; SSCC labels shipping pallets and cartons; GS1 EPCIS (Electronic Product Code Information Services) standard records traceability events (manufacture, ship, receive) in a common format. Manufacturing ERPs targeting retail supply chains (e.g., Walmart, Amazon) must support GS1 labelling and EPCIS event reporting.

---

### Security & Compliance Standards

**OWASP API Security Top 10 (2023 edition)**
- Official URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The authoritative checklist for API security vulnerabilities. A manufacturing ERP exposing REST APIs to shop floor devices, AI agents, and MES systems must address all OWASP API Security Top 10 risks: broken object-level authorisation, broken authentication, excessive data exposure, lack of rate limiting, and injection vulnerabilities. In 2026, agentic AI integration with ERP APIs has created new attack surfaces that OWASP is tracking for the next update cycle.

**NIST SP 800-82 Rev 3 — Guide to OT Security**
- Official URL: https://csrc.nist.gov/publications/detail/sp/800-82/rev-3/final
- NIST guidance for securing Operational Technology (OT) systems including industrial control systems, PLCs, and SCADA systems. Relevant to the OPC-UA and MTConnect integration layer of a manufacturing ERP: communication between shop floor devices and the ERP must meet OT security requirements including network segmentation, encrypted transport (OPC UA TLS), and authenticated device identity.

**ISO/IEC 27001:2022 — Information Security Management**
- Official URL: https://www.iso.org/standard/27001
- The international ISMS standard. Manufacturing ERPs handling production data, employee records, and customer IP (drawings, BOMs) are expected to operate under an ISO 27001-aligned security framework. Relevant for SaaS manufacturing ERP vendors seeking enterprise customers.

**GDPR (Regulation EU 2016/679)**
- Official URL: https://gdpr-info.eu/
- Manufacturing ERPs process personally identifiable information for employees (time records, job card completions, payroll-adjacent data). GDPR compliance governs data retention, right to erasure, and data processing agreements for cloud-hosted manufacturing ERPs with EU customers.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- Official URL: https://modelcontextprotocol.io/specification/2025-11-25
- The open protocol enabling AI agents to access structured data and execute actions in external systems. In 2026, Microsoft Dynamics 365 has shipped an ERP MCP server providing AI agents with governed access to finance and operations data and business logic. An AI-native manufacturing ERP should publish an MCP server exposing key manufacturing resources (BOMs, work orders, production schedules, machine status, quality records) as MCP tools and resources, enabling AI scheduling agents, quality prediction agents, and natural-language interfaces to operate against live ERP data without bespoke API integrations.
- Microsoft Dynamics 365 ERP MCP server reference: https://www.microsoft.com/en-us/dynamics-365/blog/it-professional/2025/11/11/dynamics-365-erp-model-context-protocol/
- MCP 2026 enterprise roadmap: https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/

---

## Similar Products — Developer Documentation & APIs

### ERPNext (Frappe)

- **Description:** MIT-licensed open-source manufacturing ERP built on the Frappe framework. The most capable open-source manufacturing ERP available in 2026, covering BOM, work orders, job cards, quality inspection, subcontracting, and production planning.
- **API Documentation:** https://frappeframework.com/docs/v14/user/en/api/rest (Frappe REST API Reference)
- **ERPNext Developer Docs:** https://docs.erpnext.com/ and https://github.com/frappe/erpnext/wiki/Developer-Docs
- **SDKs/Libraries:** frappe-js-sdk (JavaScript): https://github.com/The-Commit-Company/frappe-js-sdk; Python client: standard `requests` library against the REST API; community Node.js client packages available via npm
- **Developer Guide:** https://frappeframework.com/docs/v14/user/en/guides/integration/rest_api
- **Standards:** REST/JSON; all DocTypes auto-generate REST endpoints; partial OpenAPI coverage; no native OData support
- **Authentication:** API Key + API Secret token (Authorization: token api_key:api_secret); Basic auth (Base64); OAuth 2.0 bearer token
- **Notes:** ERPNext API is RESTful with resource-based URLs per DocType. Frappe's RPC layer (`/api/method/`) allows calling any whitelisted Python method. The API lacks a complete machine-readable OpenAPI spec; the community-maintained unofficial documentation at https://github.com/alyf-de/frappe_api-docs fills this gap.

---

### Epicor Kinetic

- **Description:** Commercial manufacturing ERP purpose-built for discrete manufacturers (job shop, make-to-order, engineer-to-order). Industry-leading APS scheduling and real-time capable-to-promise. Deployed as SaaS or on-premises.
- **API Documentation:** https://www.epicor.com/en/products/enterprise-resource-planning-erp/kinetic/tools-and-technology/open-rest-api/
- **Interactive REST Help:** Built-in Swagger/OpenAPI interactive help embedded in the Kinetic application; browse services, inspect metadata, and test calls from within the ERP
- **Developer Portal:** https://kinetic.developer.epicor.com/
- **SDKs/Libraries:** No official SDK; REST API is the primary integration mechanism; CData drivers available for JDBC/ODBC access (https://www.cdata.com/drivers/epicorkinetic/)
- **Developer Guide:** https://tomerlin-erp.com/epicor-erp-rest-api/ (community guide); Epicor User Help Forum: https://www.epiusers.help/
- **Standards:** REST/JSON with OData query support; OpenAPI/Swagger documentation; all ERP services exposed via the same REST service architecture used to build the product itself
- **Authentication:** OAuth 2.0; API Key; Epicor Identity Provider (IDP)
- **Notes:** Epicor's REST API exposes all ERP business objects and logic through a uniform service layer. The same REST endpoints power the Kinetic web UI, ensuring API parity with UI functionality. Epicor Connected Process Control (add-on) provides OPC-UA machine connectivity.

---

### Odoo Manufacturing

- **Description:** Open core manufacturing ERP. Community edition (GPL v3) covers BOM and work orders; Enterprise edition adds PLM, quality control, and maintenance. Well-integrated with Odoo's commercial ERP suite.
- **API Documentation:** https://www.odoo.com/documentation/18.0/developer/reference/external_api.html
- **Odoo 19 JSON-2 API (replacement for XML-RPC/JSON-RPC):** https://www.odoo.com/documentation/19.0/developer/reference/external_api.html
- **SDKs/Libraries:** Official Python client: `odoo-rpc-client` (PyPI); JavaScript: `@odoo-community/odooapiclient`; OCA REST framework module adds full REST/OpenAPI to any Odoo instance
- **Developer Guide:** https://www.odoo.com/documentation/18.0/developer.html
- **Standards:** Historically XML-RPC and JSON-RPC (both scheduled for removal in Odoo 20 / fall 2026); Odoo 17+ experimental REST API; Odoo 19 introduces JSON-2 API as the canonical replacement; OData not natively supported
- **Authentication:** API Key (Odoo 14+); Session cookie; OAuth 2.0 via third-party module
- **Notes:** Odoo is migrating its external API from XML-RPC/JSON-RPC to a new JSON-2 protocol in Odoo 19–20. Integrations built for current Odoo versions should use JSON-RPC for stability, or target the OCA REST API module for REST/OpenAPI access. Odoo IoT Box provides basic hardware I/O connectivity for shop floor devices but does not implement OPC-UA.

---

### SAP S/4HANA Manufacturing

- **Description:** Enterprise manufacturing ERP module within SAP S/4HANA. Covers PP/DS (Production Planning and Detailed Scheduling), QM (Quality Management), PM (Plant Maintenance), and Digital Manufacturing Cloud (DMC) for MES-level shop floor execution. Designed for large manufacturers ($100M+ revenue).
- **API Documentation (SAP Business Accelerator Hub):** https://api.sap.com/
- **Manufacturing-specific OData APIs:** https://api.sap.com/package/S4HANAOPAPI/odata (S/4HANA on-premise) and https://api.sap.com/package/SAPS4HANACloud/odata (S/4HANA Cloud)
- **SDKs/Libraries:** SAP Cloud SDK (Java, JavaScript/TypeScript): https://sap.github.io/cloud-sdk/; ABAP SDK for Google Cloud; Python: community `pyrfc` for BAPI/RFC access; REST is the preferred method for new integrations
- **Developer Guide:** https://developers.sap.com/; SAP Integration Suite documentation: https://api.sap.com/package/SAPIntegrationSuite
- **Standards:** OData V2 (majority of existing APIs) and OData V4 (new APIs); REST/JSON; OpenAPI documentation via the Business Accelerator Hub; SOAP for legacy integration; IDoc format for EDI-style asynchronous document exchange
- **Authentication:** OAuth 2.0 (SAP Identity Authentication Service); Basic auth (deprecated for production); mTLS for system-to-system communication
- **Notes:** SAP's manufacturing API surface is vast; the Business Accelerator Hub provides a sandbox environment with try-it-now API testing. SAP Asset Intelligence Network provides OPC-UA and MTConnect connectivity to shop floor machines. SAP Digital Manufacturing Cloud exposes its own REST API for MES data.

---

### Infor CloudSuite Industrial (SyteLine)

- **Description:** Commercial SaaS manufacturing ERP with deep discrete and process manufacturing capabilities. Pre-built vertical templates for aerospace (AS9100) and automotive (IATF 16949). Includes Infor Coleman AI for embedded demand forecasting.
- **API Documentation (Infor Developer Portal):** https://developer.infor.com/hub/apicatalog
- **CloudSuite Industrial Documentation:** https://docs.infor.com/csbi/latest/en-us/csbiolh/default.html
- **Infor OS API Gateway:** https://developer.infor.com/tutorials/api-gateway
- **SDKs/Libraries:** Infor ION API (integration middleware); REST SDK for JavaScript and Java available via the Infor Developer Portal; Infor OS provides pre-built connectors to Salesforce, AWS, Azure
- **Developer Guide:** https://developer.infor.com/
- **Standards:** REST/JSON (Infor ION REST); OData; SOAP (legacy); ION (Infor's event-driven integration bus for asynchronous document exchange); B2MML for MES integration
- **Authentication:** OAuth 2.0 (Infor Identity Provider — Ming.le / Infor OS); API Key for service accounts
- **Notes:** Infor uses ION (Intelligent Open Network) as its integration middleware, supporting event-based, API-based, file-based, and data lake integration patterns. Infor Factory Track provides OPC-UA machine connectivity for shop floor IoT integration.

---

### SYSPRO

- **Description:** Mid-market manufacturing and distribution ERP. Available as SaaS subscription or perpetual on-premises licence. Includes integrated CMMS (maintenance management) within the ERP — uncommon at its price tier.
- **API Documentation (SYSPRO Developer Portal):** https://developer.syspro.com/documentation/
- **SYSPRO Developer Hub:** https://developer.syspro.com/
- **OData Documentation:** https://help.syspro.com/syspro-8-2023/topics/insights-and-reporting/syspro-odata/syspro-odata.htm
- **Open Reporting API Reference:** https://help.syspro.com/syspro-8-2024/resources/pdfs/2022-release/topics/syspro-8-srs-api-reference-guide.pdf
- **SDKs/Libraries:** No official SDK; REST and SOAP APIs are the primary integration paths; CData drivers for ODBC/JDBC
- **Standards:** REST/JSON; OData (for reporting and data access); SOAP; SYSPRO's OData endpoint provides a RESTful interface to the SYSPRO database via HTTPS
- **Authentication:** SYSPRO credentials (operator/password); API Key for external integrations; OAuth 2.0 not natively supported in all versions
- **Notes:** SYSPRO's developer portal targets ISV and systems integration partners building on the platform. The OData interface is particularly useful for BI and reporting tool integration (Power BI, Tableau).

---

### Microsoft Dynamics 365 Supply Chain Management

- **Description:** Enterprise-grade manufacturing ERP module within Microsoft Dynamics 365. Covers production planning, manufacturing execution, quality management, asset management, and supply chain. Includes Copilot AI features for manufacturing insights and the first major ERP to ship a native MCP server (2025).
- **API Documentation:** https://learn.microsoft.com/en-us/dynamics365/supply-chain/
- **Data Integration APIs:** https://learn.microsoft.com/en-us/dynamics365/supply-chain/procurement/contract-lifecycle-management/developer/clm-data-integration-apis
- **Traceability API:** https://learn.microsoft.com/en-us/dynamics365/supply-chain/traceability/developer/traceability-api
- **Business Central API v2.0:** https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/api-reference/v2.0/ (for SMB tier)
- **MCP Server:** https://www.microsoft.com/en-us/dynamics-365/blog/it-professional/2025/11/11/dynamics-365-erp-model-context-protocol/
- **SDKs/Libraries:** Dynamics 365 Finance and Operations SDKs via NuGet; Power Platform connectors; Azure Logic Apps connectors; OData client libraries (any OData V4-compatible client)
- **Standards:** OData V4 (primary external API protocol); REST/JSON; OpenAPI via Microsoft API Business Hub; 5,000+ extensibility points via X++ and Power Platform
- **Authentication:** Azure Active Directory (Entra ID) OAuth 2.0; Service Principal authentication for system integration; MCP server uses Azure AD token-based auth
- **Notes:** Dynamics 365 Supply Chain Management is notable for being the first major ERP to ship a native MCP server (November 2025), exposing hundreds of thousands of ERP functions for AI agent access. The 2026 release wave 1 (April–September 2026) includes improvements to material picking and shop floor agility. For SMB manufacturers, Dynamics 365 Business Central provides a lighter-weight alternative with full OData V4 API coverage.

---

### Global Shop Solutions

- **Description:** US-focused manufacturing ERP for job shops and custom (make-to-order, engineer-to-order) manufacturers. Strong in quoting, shop floor data collection, and Gantt scheduling. SMB-oriented, quote-based pricing.
- **API Documentation:** https://www.globalshopsolutions.com/software-application-integrations
- **Integration Hub:** https://www.makini.io/integrations/global-shop-solutions (third-party integration catalogue)
- **SDKs/Libraries:** No publicly documented SDK; REST API available for partner integrations; EDI connectivity for OEM purchase orders
- **Standards:** REST API (limited public documentation); EDI X12 for OEM integration; CAD/CAM data import for BOM creation
- **Authentication:** Credential-based; API Key for partner integrations (documentation gated behind partner program)
- **Notes:** Global Shop Solutions does not publish a public developer portal. API access is managed through their partner programme. Third-party platforms (Makini, Flowgear) provide pre-built connectors that abstract the proprietary API.

---

## Notes

**Emerging Standard — OPC UA PubSub (IEC 62541-14:2026):** The newly published EN IEC 62541-14:2026 extends OPC UA with a Publish-Subscribe communication model, complementing the existing client-server model. PubSub enables efficient broadcast of high-frequency machine data (sensor readings, cycle counts) to multiple consumers simultaneously — including an ERP scheduler, a quality prediction model, and a digital twin — without polling overhead. AI-native manufacturing ERPs ingesting real-time shop floor data should target OPC UA PubSub rather than request-response polling for high-volume machine data streams.

**Converging standards — MTConnect and OPC UA:** The MTConnect Institute and OPC Foundation are actively maintaining and updating the joint OPC UA Companion Specification for MTConnect. The companion specification allows a single OPC UA client to consume data from both OPC UA native machines and MTConnect-enabled CNC tools, simplifying the shop floor connectivity layer of a manufacturing ERP.

**API migration risk — Odoo:** Odoo has announced removal of XML-RPC and JSON-RPC in Odoo 20 (targeted fall 2026) in favour of the new JSON-2 API introduced in Odoo 19. Integrations built against Odoo's legacy protocols must be migrated before the Odoo 20 upgrade cycle begins.

**MCP as the AI integration layer:** Microsoft Dynamics 365's MCP server (2025) signals that MCP is becoming the standard protocol for AI agent access to ERP systems. An AI-native manufacturing ERP should treat MCP server publication as a first-class deliverable alongside the REST API, enabling AI scheduling agents, BOM assistants, and quality prediction agents to be developed independently by the community using standard MCP clients.
