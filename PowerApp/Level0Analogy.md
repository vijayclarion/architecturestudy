Here's everything so far, told as one hospital. Read the story first, then use the table as your revision sheet.

## The story

**CityCare Hospital** is the whole **Power Platform**.

Inside, there are **departments** (the **products**): data-entry screens for staff (**Power Apps**), automated workers that move things along (**Power Automate**), a help desk you can ask questions (**Copilot Studio**), a public reception window (**Power Pages**), and reports on the manager's wall (**Power BI**, though it reports to its own separate management office).

Every department uses the same **building services** (the **shared services**): one **patient-records system** (**Dataverse**), **ID badges** (**Entra ID**), **phone lines** to other hospitals and insurers (**connectors**), and the **building rules and security office** (**governance**: admin center, DLP).

The hospital is divided into **wings** (**environments**). Each wing has its own rooms, its own records cabinet, its own door rules and its own regulations. The **open lobby** (**Default**) is where anyone can walk in and set something up, so you never run real patient care there. There's your **private practice room** (**Developer**), the **training wing** (**Sandbox**), the **live hospital** (**Production**), and a **temporary room** that gets removed (**Trial**). Some wings have **extra security officers and CCTV** (**Managed Environment**), and that costs extra (premium).

The hospital is built in **one country** (**region**). You can't pick it up and move it later, so you decide on day one where patient records must legally live.

New equipment always moves the same way: **practice room → training wing → live hospital** (**Dev → Test → Prod**). A doctor can prepare a change privately (**Save**), but once it's rolled out (**Publish**), everyone gets it. That's why it gets tested in the training wing first.

**Staff** use internal screens (**canvas apps**) from any device: a **terminal in the browser** (**web player**) or a **handheld device** (**Power Apps mobile app**). To use a screen, a staff member needs two things: to be **given access to it** (**sharing**) and a **paid staff badge** (**licence**). The badge proves **who you are** (**authentication, Entra ID**). The **door access list** decides **which rooms you can enter** (**authorization, Dataverse security roles**).

**Patients and visitors** come through the **public reception window** (**Power Pages**). Some areas are open to anyone (**anonymous**), and some need a patient ID (**authenticated**). Visitors don't get staff badges. The hospital buys **visitor passes in bulk packs** (**capacity packs per website per month**), which is much cheaper per person.

**Basic facilities** like corridors and the canteen come free with employment (**seeded M365 rights, standard connectors**). The **specialist wing** with the records system costs extra per staff member (**premium: Dataverse, premium connectors, custom connectors**).

The building has **utility allowances**: so much water and electricity per day (**request limits per licence per 24 hours**). If everyone runs the taps at once, the pressure drops (**throttling, error 429**). A clerk who can't search the archive properly only checks the **first 500 files** in the cabinet (**delegation limit, 500 by default, 2,000 max**). Bulky items like X-ray film go to **external storage**, not the records cabinet (**files in SharePoint or Blob storage, not Dataverse**).

Fittings **built into the walls** stay behind if you move hospitals (**lock-in: screens, Power Fx, flows, topics**). Valuable equipment is kept **on wheels** or **outsourced to a shared lab** that every hospital can use (**an external API through a custom connector**).

Finally, a hospital is great for patient care, but you don't run a **car factory** inside it (**not the core system for extreme volume, low latency, consumer-grade UX or heavy computation**). The factory stays separate, and the hospital connects to it.

## Revision table

| Concept | Hospital analogy | Remember for the exam |
|---|---|---|
| Power Platform | The whole hospital | Low-code, not no-code |
| Products | Departments | Choosing a product = user-experience decision |
| Shared services | Building services | The hard, important decisions live here |
| Dataverse | Patient-records system | Premium; one per environment at most |
| Entra ID | ID badge | Who you are (authentication) |
| Security roles | Door access list | What you can do (authorization) |
| Connectors | Phone lines to other organizations | Standard (free with M365) vs premium |
| Environment | A wing | Container + boundary for security, region, DLP |
| Default | Open lobby | Never for business apps |
| Developer | Private practice room | Single user, never production |
| Sandbox | Training wing | Team dev, test, UAT; copy and reset |
| Production | Live hospital | What the business depends on |
| Trial | Temporary room | Testing a *platform feature*, expires |
| Managed Environment | Wing with security office and CCTV | Premium governance features |
| Region | Country the building is in | Choose on day one; data residency |
| Dev → Test → Prod | Practice room → training wing → live hospital | Changes move one way |
| Save vs Publish | Prepared privately vs rolled out to everyone | Users see only the published version |
| Canvas app | Staff screens | Web player, mobile app, Teams, SharePoint |
| Sharing + licence | Given access + paid badge | Both are needed |
| Power Pages | Public reception window | External users; table permissions and web roles |
| Power Apps Premium | Paid staff badge | $20/user/month (list), unlimited apps |
| Pay-as-you-go | Day pass | $10 per active user per app per month |
| Power Pages packs | Bulk visitor passes | Packs of 100 authenticated / 500 anonymous |
| Seeded rights | Free corridors and canteen | M365 + standard connectors only |
| Request limits | Daily utility allowance | Per licence per 24 hours |
| Throttling | Water pressure drops | Error 429 |
| Delegation | Clerk only checks the first 500 files | 500 default, 2,000 max |
| Lock-in | Fittings built into the walls | Screens, Power Fx, flows, topics |
| Portable logic | Equipment on wheels / shared lab | External API + custom connector |
| Poor fit | A car factory inside a hospital | Keep the core elsewhere; connect to it |

Shall we continue with **Level 0, Topic 3**? Also, you haven't answered yet: should I apply the curriculum changes (a Copilot Studio basics lesson before PL-900, Level 9 made optional, and Foundry and Dynamics 365 added for the later exams)?