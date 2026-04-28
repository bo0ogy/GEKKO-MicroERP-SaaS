# GEKKO-Entreprise: Invoice Logic & Integrity (CRITICAL)

Because an ERP deals with legal and financial compliance (especially strict Moroccan tax rules and chronological sequencing), invoice creation is the most sensitive operation in the application.

## 1. The Creation Flow (Request → DB)

When a user clicks "Create Invoice" on the frontend, the data follows a strict, zero-trust path:

1. **Frontend Request:** The frontend sends a `POST /api/invoices`. 
   * **CRITICAL RULE:** The frontend *only* sends intentions: `client_id`, an array of `items` containing `product_id`, `quantite`, and `discount_amount`. 
   * The frontend **never** sends the `total_ttc` or `montant_tva`. 
2. **Validation (DTO):** The `CreateInvoiceDto` intercepts the request. It verifies that `quantite` is a positive number, that `discount_amount` is not negative, and that the `client_id` is a valid UUID.
3. **Business Logic (Service):** The `InvoicingService` takes over.
   * **Verification:** It queries the database to verify the `client_id` actually belongs to the user's `organization_id` (Tenant Isolation).
   * **Price Fetching:** It queries the DB for the true `prix_unitaire_ht` and `TaxRate` of every `product_id`. This prevents a malicious user from modifying the frontend code to send a product with a 0 DH price.
4. **Calculation Phase:** The backend securely calculates the math (see section below).
5. **Transaction Execution:** The backend opens a strict PostgreSQL transaction, assigns the chronological number, saves the data, and commits.

## 2. How and Where Totals are Calculated

**Where?** Purely in the **Backend (InvoicingService)**.
* The frontend *does* calculate totals, but strictly for visual UI purposes.
* The database *stores* the totals, but it does not run the math.

**How?**
To prevent inconsistent totals (like rounding errors where `Line 1 + Line 2` does not exactly match the `Global Total`), we follow these rules:
* We never use standard JavaScript floats (which cause `0.1 + 0.2 = 0.30000000000000004`). We use a library like `decimal.js`, or we store all monetary values in **cents** (integers) in the database and divide by 100 on the frontend.
* The math follows a strict order:
  1. `Line_HT = (prix_unitaire_ht * quantite) - discount_amount`
  2. `Line_TVA = Line_HT * (TaxRate / 100)`
  3. `Line_TTC = Line_HT + Line_TVA`
  4. Finally: `Global_TTC = Sum(All_Line_TTC)`

## 3. Preventing Partial Writes & Transaction Strategy

A partial write happens if the Node.js server crashes or loses database connection halfway through saving an invoice. You would end up with an Invoice Header in the DB, but zero items attached to it. 

We completely eliminate this risk using **PostgreSQL ACID Transactions**.

### The Transaction Strategy (Step-by-Step)
Inside our NestJS Service, we wrap the entire operation in a `prisma.$transaction`.

1. **BEGIN TRANSACTION**
2. **Lock the Sequence:** We execute a raw query: `SELECT next_invoice_number FROM invoice_sequences WHERE organization_id = 'org-123' FOR UPDATE;`
   * *Why?* This creates a **Row-Level Lock**. If User A and User B click "Generate" at the exact same millisecond, PostgreSQL forces User B to wait in a queue until User A finishes. This is how we guarantee the strict legal chronology (FA-2024-001, FA-2024-002) required by Moroccan law.
3. **Insert the Header:** Create the `Invoice` row with the reserved number.
4. **Insert the Lines:** Insert the array of `InvoiceLine` rows using `createMany`.
5. **Increment Sequence:** Update the sequence table to the next number.
6. **COMMIT TRANSACTION**

**What if it crashes at step 4?**
If the Node.js server crashes at Step 4, the transaction is never committed. The database session drops, and PostgreSQL instantly **ROLLS BACK** everything. 
* The Invoice Header vanishes as if it never existed.
* The sequence lock is released without incrementing.
* The next user will correctly get FA-2024-001. No ghost data, no missing numbers in the accounting ledger.
