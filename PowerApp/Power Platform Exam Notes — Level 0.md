# Power Platform Exam Notes — Level 0

Oct 28, 2026 · @Vijay

## How to use these notes

These notes cover Level 0, Lesson 1 and the questions we discussed. Each topic has the same five parts: **Purpose**, **Key points**, **Rules**, **Terms to remember** and **Example**. Most of this content maps to the PL-900 skills on Power Platform business value, environments and licensing.

Prices are US list prices in USD as of October 2026. They vary by country and agreement and change often, so check the official Power Apps and Power Pages pricing pages before an exam or a client quote.

## Topic 1: What Power Platform is

**Purpose:** A set of Microsoft low-code products for building business apps, automation, agents, websites and reports, all running on the same shared services.

**Key points**

- Products are what people build with and use. Shared services sit underneath and are reused by every product.
- The hard-to-reverse decisions (data, security, DLP, ALM, licensing) live in the shared services, not in the choice of product.
- It is low-code, not no-code: professional developers can extend it with code, APIs and Azure.
- Power BI belongs to the family but is managed through its own admin portal and workspaces, not Power Platform environments.

| Product | Purpose |
| --- | --- |
| Power Apps | Business apps: canvas apps (you design the screens) and model-driven apps (screens generated from the data model) |
| Power Automate | Workflows: cloud flows, approvals, and desktop flows (RPA) |
| Copilot Studio | AI agents that answer from knowledge and take actions |
| Power Pages | Websites for external users |
| Power BI | Reports and dashboards |

| Shared service | Purpose |
| --- | --- |
| Microsoft Entra ID | Identity: every user, sign-in and permission |
| Dataverse | Managed data platform: tables, relationships, logic and security |
| Connectors | Prebuilt adapters to other systems; custom connectors for your own APIs |
| Power Fx | Excel-like formula language used across products |
| Environments | Containers for apps, flows, agents and data |
| Governance | Admin center, DLP policies, Managed Environments, solutions and pipelines |

**Rules**

- Choosing a product is mostly a user-experience decision. Shared-service choices decide security, integration and scale.
- Always list Entra ID among the shared services. Identity is the foundation.

**Terms to remember:** low-code, connector, custom connector, Dataverse, Power Fx, Entra ID, environment, DLP policy.

**Example:** A health insurer builds a claims app (Power Apps), an approval flow (Power Automate) and a member help agent (Copilot Studio). All three read the same Dataverse claims table and follow the same security roles. Fix the security once, and all three are fixed.

## Topic 2: Environments

**Purpose:** An environment is a container that holds apps, flows, agents and optionally one Dataverse database. It is also a boundary for security, region and DLP policies.

**Key points**

| Type | What it is | Use it for |
| --- | --- | --- |
| Default | A special production environment created automatically for every tenant; everyone in the organization can build in it by default | Personal productivity only, never business-critical apps |
| Production | Supported environment for live systems | Anything the business depends on |
| Sandbox | Nonproduction environment with copy and reset features | Development, test, UAT |
| Developer | Single-user environment from the free Developer Plan; can't run or share production apps; stays available only while you actively use it | Learning and individual building |
| Trial | Short-term evaluation environment | Trying features |

**Rules**

- Separate dev, test and production for any real project.
- Never run business-critical apps in the default environment.
- Choose the region at creation for data residency. You can't easily change it later.
- Developer environments are never used for go-live: single user, no production use, one owner.
- Settings that differ between environments (API URLs, connections) are handled with environment variables and connection references (Level 7).

**Terms to remember:** default environment, production, sandbox, developer environment, region, Dataverse database, Managed Environment, Power Platform admin center.

**Example:** A hospital project with four developers uses developer environments for experiments, a shared sandbox for integration, a sandbox for UAT, and a production environment in an India region with Dataverse, strict DLP, and only managed solutions deployed.

## Topic 3: Canvas apps vs Power Pages

**Purpose:** Both put a user interface on your data. The difference is who the user is: canvas apps are for internal staff, Power Pages is for people outside the organization.

**Key points**

|  | Canvas app | Power Pages |
| --- | --- | --- |
| Users | Internal staff | External: patients, policyholders, customers, partners |
| What it is | An app running inside a Microsoft host | A website with its own public URL (custom domain possible) |
| Where it runs | Power Apps web player in a browser, Power Apps mobile app, Teams, SharePoint | Any web browser |
| Sign-in | Work account (Microsoft Entra ID) | External sign-in (email, social accounts) or anonymous |
| Data security | Dataverse security roles | Table permissions and web roles |
| Licensing | Per user | Capacity per website per month |

**Rules**

- A canvas app is not "mobile only". Build once; it runs on laptop, phone and tablet. Only the layout (phone, tablet, responsive) differs.
- A user can open a canvas app only when (1) it is shared with them and (2) they have the right licence.
- On mobile, users install the Power Apps mobile app once; every app shared with them appears inside it. No App Store publishing needed.
- User outside your Entra ID tenant → think Power Pages first. Staff → think Power Apps.
- Every Power Pages site touching sensitive data needs carefully designed table permissions. A mistake exposes one patient's data to another.

**Terms to remember:** Power Apps web player, Power Apps mobile app, phone/tablet/responsive layout, sharing, authenticated user, anonymous user, table permissions, web roles.

**Example:** CityCare Pharmacy staff track refills in a canvas app, opened in the browser on store laptops and in the Power Apps mobile app on tablets. Patients use a Power Pages site: opening hours and FAQ are anonymous; "My refill status" requires sign-in.

## Topic 4: Licensing

**Purpose:** Licensing decides what each user and each site costs. Design choices (Dataverse, premium or custom connectors) directly change the price, so architects calculate it in every design.

**Key points: Power Apps (internal users)**

First question: does the app use anything premium? Standard connectors only (SharePoint, Outlook, Excel, Teams) → covered by Microsoft 365 business licences ("seeded" rights). Dataverse, SQL, custom or premium connectors → every user needs a premium licence.

| Plan | Price (USD list) | Best for |
| --- | --- | --- |
| Power Apps Premium | $20 per user per month, unlimited apps; $12 at 2,000+ new licences | Daily users, or users of several apps |
| Pay-as-you-go | $10 per active user per app per month, billed through Azure; no charge if the user doesn't open the app that month | Occasional or seasonal use |
| Per app ($5 per user per app) | No longer on Microsoft's public price list for new customers since 2 January 2026; still available through some partner channels and Enterprise Agreements | Existing customers |

Break-even: two apps on pay-as-you-go ($10 × 2) equals one Premium licence ($20). A reported increase to $22 (and $14 at volume) from 1 January 2027 has been seen in one third-party source only; confirm before relying on it.

**Key points: Power Pages (external users)**

| Licence | How it's sold | Launch price (2022, USD) |
| --- | --- | --- |
| Authenticated users per website | Packs of 100 users, one website, one calendar month | $200 per pack; pay-as-you-go $4 per user |
| Anonymous users per website | Packs of 500 users, one website, one calendar month | $75 per pack; pay-as-you-go $0.30 per user |

Capacity is assigned at the environment level. Volume tiers lower the price per pack.

**Rules**

- Calculate licensing cost in every design, not after building.
- Moving data from SharePoint to Dataverse turns a free app into a premium one for every user.
- External users go on Power Pages, not Power Apps: a few dollars per visitor vs $20 per named user.
- Managed Environments are a premium feature with their own licensing requirements.
- The Power Platform Licensing Guide (PDF) is the document architects rely on.

**Terms to remember:** seeded licensing, premium connector, Power Apps Premium, pay-as-you-go, active user, authenticated user, anonymous user, capacity pack, Licensing Guide.

**Example:** CityCare: 200 staff on a Dataverse app = 200 × $20 = about $4,000 per month. 300 patients signing in to the Power Pages site in a month = 3 authenticated packs = about $600 per month at launch pricing. Using SharePoint instead of Dataverse would remove the $4,000 but lose Dataverse security and scale.

## Topic 5: Platform limits

**Purpose:** Power Platform is shared cloud infrastructure, so Microsoft limits how much any user, app or tenant can consume. Treat limits as design constraints, like API rate limits on any SaaS.

**Key points**

| Limit | What it means | Number to remember |
| --- | --- | --- |
| Request limits (entitlement) | Every call from an app or flow to Dataverse or a connector counts as a request, allowed per licence per 24 hours | 40,000 per paid user licence per 24 hours (Microsoft's example); add-ons give +50,000 each |
| Service protection limits | Protect against sudden surges; requests are rejected when exceeded | Error 429 "Too Many Requests" |
| Delegation | A non-delegable canvas app formula only processes the first rows locally | 500 rows by default, maximum 2,000 |
| Storage capacity | Dataverse storage is split into database, file and log; extra costs money | Check the licence entitlements |

**Rules**

- Process records in batches, not one call at a time.
- Don't loop over thousands of Dataverse records inside a flow.
- Move heavy bulk processing to Azure (Azure Functions, Data Factory).
- Store large files in SharePoint or Azure Blob Storage, not Dataverse.
- Use delegable formulas when searching large tables.

**Terms to remember:** Power Platform request, entitlement limit, service protection limit, 429 error, throttling, delegation, data row limit, capacity add-on, database/file/log capacity.

**Example:** An insurer's canvas app searches 300,000 claims with a non-delegable formula, so only the first 500 rows are searched and users report "missing" claims. A nightly import of 2 million claims runs in Azure Data Factory, not Power Automate, to avoid hitting request limits.

## Topic 6: Lock-in and portable logic

**Purpose:** Lock-in is how hard and expensive it is to leave the platform. Clients accept it in exchange for faster delivery; the architect makes it a conscious, documented decision.

**Key points**

| Hard to move (rebuild if you leave) | Easier to move |
| --- | --- |
| Canvas screens, Power Fx formulas | Data in Dataverse (standard OData REST API, export to Azure or Fabric) |
| Power Automate flows | Logic in your own external APIs |
| Copilot Studio topics | Documented data model |
| Business rules, Dataverse security design |  |

**Rules**

- Logic used by several systems, inside and outside Power Platform, goes in an external API: Azure Functions behind Azure API Management, called through a custom connector.
- Be precise: Dataverse has a feature called **Custom API**. It runs inside Dataverse, so it doesn't reduce lock-in. Say "external API via custom connector" for the portable pattern.
- State the trade-offs: custom connectors are premium, Azure adds hosting and operations cost, the API needs Entra ID security, and each call adds a network hop.
- Record the lock-in decision in an architecture decision record.

**Terms to remember:** lock-in, exit strategy, custom connector, Dataverse Custom API, Azure Functions, Azure API Management, OData, architecture decision record.

**Example:** CityCare's drug-interaction algorithm is built as an Azure Functions API behind API Management, exposed through a custom connector. The pharmacy app, a future website and other systems all use the same tested logic. The screens and flows stay in Power Platform, and that lock-in is accepted for faster delivery.

## Topic 7: When Power Platform fits and when it doesn't

**Purpose:** Know where the platform is the right core and where it should only be a layer around another system.

**Key points**

| Good fit | Poor fit as the core system |
| --- | --- |
| Internal business processes and "forms over data" apps | Very high-volume transaction processing |
| Approvals and workflows | Very low-latency requirements |
| Connecting systems that don't talk to each other | Consumer-grade, pixel-perfect user experiences |
| Replacing spreadsheet-and-email processes | Heavy computation |

**Rules**

- For poor-fit workloads, keep the core in a pro-code or specialized system and use Power Platform around it (intake, exceptions, approvals).
- Low-code still needs architecture: data model, ALM, security and governance.
- Common mistakes: production apps in the default environment, no architecture, SharePoint lists as a relational database, discovering licensing cost after building.

**Terms to remember:** forms over data, system of record, pro-code, shadow IT, governance.

**Example:** An insurer's core claims adjudication engine stays in its existing system. Power Platform handles claims intake, exception handling and high-value approvals around it.

**Model answer:** "Power Platform fits process and workflow applications. For extreme volume, latency, complex consumer UX or heavy compute, I keep the core in a pro-code or specialized system and use Power Platform around it."

## Glossary and exam traps

**Glossary**

| Term | Meaning |
| --- | --- |
| Dataverse | Managed data platform with tables, relationships, logic and security; premium |
| Connector | Prebuilt adapter to another system; standard or premium |
| Custom connector | Your own connector wrapping your API; premium |
| Environment | Container for apps, flows, agents and data; boundary for security, region and DLP |
| DLP policy | Rules on which connectors can be used together |
| Managed Environment | Premium governance features applied to an environment |
| Solution | Package of components moved between environments (Level 7) |
| Seeded licensing | Power Apps rights included in Microsoft 365, standard connectors only |
| Delegation | Sending query work to the data source instead of processing rows in the app |
| Table permissions / web roles | How Power Pages controls which data external users see |

**Exam traps**

1. "Canvas apps are for mobile" → wrong. They run in browser, mobile app, Teams and SharePoint.
2. "Microsoft 365 users can use any Power App" → only apps using standard connectors.
3. "Developer environments can host a small production app" → no, never for production.
4. "Default environment is fine for live apps" → no, everyone can build there.
5. "Custom API = external API" → no, Dataverse Custom API runs inside Dataverse.
6. "Power BI lives in Power Platform environments" → no, it uses its own admin portal and workspaces.
7. "Search shows all rows" → not with non-delegable formulas: 500 by default, 2,000 max.

**Sources**

- [Create a developer environment](https://learn.microsoft.com/power-platform/developer/create-developer-environment)
- [Environments overview](https://learn.microsoft.com/ga-ie/power-platform/admin/environments-overview)
- [ALM basics](https://learn.microsoft.com/power-platform/alm/basics-alm)
- [Requests limits and allocations](https://learn.microsoft.com/en-us/power-platform/admin/api-request-limits-allocations)
- [Dataverse API limits overview](https://learn.microsoft.com/en-us/power-apps/maker/data-platform/api-limits-overview)
- [Power Pages pricing](https://www.microsoft.com/en-us/power-platform/products/power-pages/pricing)
- [Power Pages launch licensing announcement](https://www.microsoft.com/licensing/news/power_pages_general_availability)
- [Power Apps per app plan changes (third-party)](https://samexpert.com/power-apps-per-app-plan-retired/)
- [Power Apps licensing explained (third-party)](https://lowcodestacks.com/guides/power-apps-licensing-explained)
