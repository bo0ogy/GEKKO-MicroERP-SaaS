# GEKKO Micro-ERP: UBL 2.0 SaaS Pipeline

A multi-tenant SaaS data pipeline and Micro-ERP designed strictly to comply with the **UBL 2.0** and the **Moroccan General Directorate of Taxes (DGI)** electronic invoicing standards.

## 🚀 Architecture & Tech Stack

This project is built as a highly structured, scalable enterprise application using modern backend standards:
- **Backend Framework:** NestJS (Modular, TypeScript-first API)
- **Database Engine:** PostgreSQL
- **ORM & Data Isolation:** Prisma (Utilizing Prisma Middleware for strict `organizationId` multi-tenant isolation)
- **Concurrency Control:** Row-Level Locking (`prisma.$transaction`) to guarantee legal sequential invoice numbering.
- **Frontend:** Next.js (utilizing TailwindCSS and modern UI templates)

## 📂 Repository Structure

```text
├── docs/                             # Core project documentation and compliance guidelines
│   ├── development_guide.md          # Strict NestJS/Prisma architectural rules and deployment pipeline
│   ├── database_model.md             # Core PostgreSQL schemas
│   ├── scrum_backlog.md              # Agile sprints and user stories mapping
│   ├── Charte d_incubation...pdf     # Incubation charter
│   ├── Plan Comptable marocain.pdf   # Moroccan Chart of Accounts
│   ├── loi 09-08.pdf                 # Moroccan Data Protection Law compliance
│   └── src/                          # Frontend UI templates and design assets
├── backend/                          # (Sprint 0) NestJS backend infrastructure
└── frontend/                         # (Sprint 0) Next.js frontend infrastructure
```

## 🔐 Security & Compliance

This Micro-ERP ensures absolute compliance with local Moroccan financial laws:
- **Data Privacy:** Architecture aligned with *Loi 09-08* regarding the protection of personal data.
- **Accounting Standards:** Financial logic mapped directly to the *Plan Comptable Marocain*.
- **Multi-Tenancy:** Absolute tenant data isolation implemented at the ORM layer to prevent cross-tenant data leaks.
- **Transactional Integrity:** Bulletproof invoice generation using database-level locking to prevent numbering conflicts during concurrent user actions.

## ⚙️ Development Guidelines
Before contributing, all developers **must** read the `docs/development_guide.md` to understand the Docker Compose setup, NestJS architecture, and Prisma migration standards.

---
*Architected for the Moroccan TPE Market by Hamza Benzina.*
