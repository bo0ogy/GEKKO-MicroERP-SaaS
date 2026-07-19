# GEKKO-Entreprise: Database Design Integration

## 1. ORM vs Raw SQL

**ORM Choice:** We are using **Prisma ORM** coupled with PostgreSQL.

**Why Prisma over Raw SQL?**
* **Type Safety:** In an ERP, data integrity is everything. Prisma auto-generates a highly strict TypeScript client based on our database schema. If you try to save a string in a `prix_unitaire_ht` float column, TypeScript will throw a compilation error before the code even runs. Raw SQL is prone to runtime errors and typos.
* **Developer Velocity:** Writing raw SQL for standard CRUD operations (creating clients, updating quotes) takes too much time. We need to respect the Agile 1-2 week Sprint cycle outlined in your backlog.
* **Hybrid Approach:** For 95% of the application, we use the ORM. For the complex 5% (like Sprint 5's massive Business Intelligence reports or heavy TVA CSV exports), Prisma allows us to use `prisma.$queryRaw` to write highly optimized pure SQL.

## 2. Mapping Backend to PostgreSQL Schema

The entire database model defined in your `docs/database_model.md` maps directly to a single declarative file in the backend: `prisma/schema.prisma`. 

For example, your Organization entity maps like this:
```prisma
model Organization {
  id          String    @id @default(uuid())
  nom_social  String
  ice         String    @db.VarChar(15) // Enforcing the 15 char limit
  
  // Relationships mapped cleanly
  clients     Client[]
  invoices    Invoice[]
}
```
This acts as the single source of truth for both the database and the backend code.

## 3. How We Handle Key Database Concepts

### A. Migrations
Prisma handles migrations elegantly. When we change the `schema.prisma` (e.g., adding the new `discount_amount` field to `QuotationLine`), we simply run:
`npx prisma migrate dev --name add_discount_amount`

Prisma instantly reads the schema, compares it to the live PostgreSQL database, and automatically generates the precise raw SQL `ALTER TABLE` file, placing it in a `prisma/migrations/` folder. This ensures that when we deploy to production, the production database safely runs those exact SQL commands in perfect sequence without losing data.

### B. Relationships (1-N, N-N)
PostgreSQL strictly enforces the foreign keys under the hood, but Prisma makes querying them effortless in the code. 
* **1-N (One-to-Many):** One `Invoice` has many `InvoiceLine`. 
* **Querying Data:** If the frontend needs an invoice with all its lines and the client details, we do not need to write 3 separate queries or complex SQL JOINs. We simply write:
  ```typescript
  const invoice = await prisma.invoice.findUnique({
    where: { id: invoiceId },
    include: {
      client: true, // Auto-fetches the client data
      lines: true,  // Auto-fetches all related lines
    }
  });
  ```

### C. Transactions (Crucial for Invoices)
**This is one of the most critical parts of the ERP.** 
When a commercial converts a Quote to an Invoice (US 4.1), we must create the `Invoice` (Header) and multiple `InvoiceLine` items. If the server crashes or the database disconnects *after* the Header is created but *before* the Lines are created, we end up with a corrupted, empty invoice in the database.

To prevent this, we use **Transactions**.
```typescript
await prisma.$transaction(async (prisma) => {
  // 1. Create the Header
  const invoice = await prisma.invoice.create({ data: {...} });

  // 2. Create the Lines linked to the Header
  await prisma.invoiceLine.createMany({ data: lines });

  // 3. Mark Quote as Invoiced
  await prisma.quotation.update({ where: { id: quoteId }, data: { status: 'Facturé' }});

  // If ANY of this fails, PostgreSQL instantly rolls back the ENTIRE operation.
  // Nothing is saved unless EVERYTHING succeeds perfectly.
});
```

**Ensuring Legal Chronology (US 4.2):**
To guarantee that invoice identifiers (FA-2024-001, FA-2024-002) follow each other strictly without gaps, we will use a combination of PostgreSQL Transactions and **Row-Level Locking** (the equivalent of `SELECT ... FOR UPDATE`). This ensures that if two users click "Create Invoice" at the exact same millisecond, the database puts one user on hold, issues FA-2024-001, and then gives the other user FA-2024-002. No duplicates, no missing numbers.
