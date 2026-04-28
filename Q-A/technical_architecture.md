# GEKKO-Entreprise: Technical Architecture Proposal

This document outlines the strategic proposal for the architecture and technology stack of GEKKO-Entreprise, tailored specifically to the requirements of the Micro-ERP SaaS for Moroccan TPEs.

## 1. Backend Framework & Technology Stack

**Recommended Stack:** Node.js with NestJS (TypeScript) + PostgreSQL + Prisma ORM

While frameworks like Django (Python) or Laravel (PHP) are excellent, NestJS is arguably the best fit for this specific modern SaaS ERP for several reasons:

* **Strict Type Safety (TypeScript):** When dealing with financial calculations (TVA, TTC, Remises) and strict legal chronologies (FA-2024-001), JavaScript's loose typing is a risk. TypeScript ensures we don't accidentally add a string to a number, preventing catastrophic accounting bugs.
* **Built-in Modularity:** NestJS forces a modular architecture out-of-the-box. The project's Sprints map perfectly to NestJS modules (`AuthModule`, `CRMModule`, `InvoicingModule`).
* **Elegant Multi-Tenancy:** We can use Prisma ORM (or TypeORM) with "Global Scopes" or middleware to automatically append `WHERE organization_id = current_user.org_id` to **every single database query**. This guarantees data isolation and prevents data leaks between companies, fulfilling the core requirement of US 1.1.
* **Full-Stack JavaScript:** If choosing React, Vue, or Next.js for the frontend, using Node.js allows us to share data transfer objects (DTOs) and types between the backend and frontend, drastically speeding up development.

## 2. Monolithic vs. Modular Service Structure

**Recommended Structure:** A Modular Monolith.

We should **avoid Microservices** for this phase of the project, but we should also **avoid a tangled "Spaghetti" Monolith**.

**The Tradeoffs:**
* **Microservices (The Bad for now):** Building separate services for Auth, Invoicing, and CRM means dealing with distributed transactions (what if a Quote is created but the Client service fails?), complex deployments, and network latency. For a startup targeting TPEs and aiming for 1-2 week sprints, microservices will negatively impact velocity.
* **Modular Monolith (The Sweet Spot):** We will build a single application and a single PostgreSQL database. However, the code will be strictly divided into independent domains (Modules).
  * *Tradeoff:* It is much faster to develop, deploy, and test than microservices.
  * *Tradeoff:* It enforces clean architecture, meaning that if GEKKO becomes massive, it is very easy to extract the "Invoicing" module into its own microservice because the code is already decoupled.

## 3. Scalability and Future Modules (Inventory, HR)

### A. Handling Scalability (Volume & Performance)
* **Stateless Architecture:** Because we are using JWT for authentication (US 1.2), the backend will not store session data in server memory. This means we can scale *horizontally* instantly. If traffic spikes at the end of the month when TPEs generate their invoices, we can spin up multiple backend servers behind a load balancer seamlessly.
* **Background Processing (Queueing):** Generating PDFs (US 3.4) and massive CSV TVA Exports (US 5.2) are CPU-heavy tasks. The API will not do this synchronously. We will use a queue (like Redis + BullMQ). The user clicks "Generate Export", the API responds immediately with "Processing...", and a background worker handles the heavy lifting, ensuring the application never slows down for other users.
* **Row-Level Security (RLS):** For the database, PostgreSQL handles massive scale beautifully. We can also leverage Postgres RLS policies as a secondary layer of defense to ensure a tenant can *never* query another tenant's rows, even if a developer makes a mistake in the application code.

### B. Integrating Future Modules (Inventory, HR)
* **Event-Driven Internal Architecture:** When adding "Inventory", we don't want to rewrite the "Invoicing" module. Instead, we will use internal events.
  * *Example:* When an invoice is validated, the `InvoicingModule` simply emits an event: `InvoiceValidatedEvent`.
  * The new `InventoryModule` listens for this event and automatically deducts the stock. The Invoicing module doesn't even need to know the Inventory module exists.
* **Plug-and-Play Structure:** Thanks to the Modular Monolith structure, adding an HR module is as simple as creating `src/modules/hr`. It will inject the core `Organization` and `User` services but will have its own dedicated database tables (e.g., `hr_employees`, `hr_leaves`) linked by `organization_id`.
