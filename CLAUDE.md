# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run the CAP server locally (with hot-reload)
cds watch

# Run with the Fiori UI open in browser
npm run watch-incidents

# Run tests
npm test

# Build for production deployment
npx cds build --production

# Deploy to BTP (requires MTA build tools)
mbt build && cf deploy mta_archives/incident-management_1.0.0.mtar
```

## Architecture

This is a **SAP Cloud Application Programming Model (CAP)** Node.js project for incident management, deployable to SAP BTP (Business Technology Platform).

### Layer overview

- **`db/schema.cds`** — Domain model: `Incidents`, `Customers`, `Addresses`, plus `Status` and `Urgency` code lists
- **`srv/services.cds`** — Two OData v4 services:
  - `ProcessorService` (requires `support` role) — for support staff; incidents use draft choreography (`@odata.draft.enabled`)
  - `AdminService` (requires `admin` role) — full CRUD on customers and incidents
- **`srv/services.js`** — Custom handlers for `ProcessorService`: urgency auto-elevation when title contains "urgent", closed-incident update guard, and customer caching from S/4HANA
- **`srv/remote.cds`** — Projection of the SAP S/4HANA `API_BUSINESS_PARTNER` external API into a local `RemoteService` with renamed fields
- **`app/incidents/`** — SAP Fiori Elements UI app (SAPUI5/OData); annotations in `app/incidents/annotations.cds` drive the list report + object page layout
- **`app/services.cds`** — Re-exports app-level annotations

### External integrations

- **`API_BUSINESS_PARTNER`** — S/4HANA OData v2 service accessed via SAP Cloud SDK. In production, resolved via a BTP Destination named `businessPartner`. Customer data is fetched on incident create/update and cached locally in the `Customers` table.
- **`NORTHWIND_LOCAL_DEST`** — Direct URL to Northwind OData service (for local testing)
- **`NORTHWIND_BTP_DEST`** — Northwind accessed via BTP Destination named `northwind`

### Authentication

- **Development**: Mocked auth via `package.json` `cds.requires` config. Test users: `alice` (admin + support), `bob` (support), `incident.support@tester.sap.com` (support, password: `initial`)
- **Production**: XSUAA (`xsuaa` service on BTP). Roles defined in `xs-security.json`: `support` and `admin`

### Deployment (BTP)

`mta.yaml` defines the multi-target application:
- `incident-management-srv` — Node.js CAP backend
- `incident-management-db-deployer` — HDI container deployer for SAP HANA
- `incidentmanagementincidents` — HTML5 app deployer for the Fiori UI
- BTP services required: XSUAA (`xsuaa`), Destination, HANA HDI container, HTML5 Apps Repo

### Data

Seed CSV files are in `db/data/` using the naming convention `<namespace>-<EntityName>.csv`.

## Developer notes

- Use "" for strings