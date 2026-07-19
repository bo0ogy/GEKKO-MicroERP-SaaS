# GEKKO-Entreprise: Authentication & Security

Security is non-negotiable for an ERP handling sensitive financial data, client lists, and corporate identities. Here is the exact strategy we will implement to secure the application.

## 1. How Will Users Authenticate?

We will use **Stateless JWT (JSON Web Tokens)**.

1. **Login:** A user sends their email and password. The backend hashes the password and compares it to the database using `bcrypt`.
2. **Token Generation:** If successful, the server generates a cryptographically signed JWT. Crucially, the payload of this token contains:
   * `sub` (User ID)
   * `organization_id` (The Tenant ID)
   * `role` (e.g., "Admin", "Comptable")
3. **Subsequent Requests:** The frontend stores this token and attaches it to the `Authorization: Bearer <token>` header of every single API request.
4. **Why Stateless?** The backend does not need to look up the database for the user session on every request. It simply verifies the cryptographic signature of the token. This makes the API extremely fast and easy to scale.

## 2. Managing Roles and Permissions (RBAC)

We will implement **Role-Based Access Control (RBAC)**. 

### The Roles
* **Admin:** Has full control (billing, adding users, changing accounting regimes).
* **Commercial (Sales):** Can manage clients, products, create quotes, and convert them to invoices. *Cannot* modify global tax configurations or export VAT logs.
* **Comptable (Accountant):** Can view all invoices, register payments, and generate CSV/TVA exports. *Cannot* create quotes or modify the product catalog.

### How it's enforced in the code
NestJS makes this elegant using custom Decorators and Guards.
If an accountant tries to create an invoice, they will hit a route protected like this:

```typescript
@Post()
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('Admin', 'Commercial') // Only these roles are allowed
async createInvoice(@Body() dto: CreateInvoiceDto) {
  // If a 'Comptable' tries to access this, the RolesGuard intercepts 
  // the request and returns a 403 Forbidden instantly.
}
```

## 3. Protecting the API

To protect the server from attacks, we implement the following layers:
* **Rate Limiting:** We use `@nestjs/throttler` to block Brute Force attacks. (e.g., maximum 5 login attempts per IP per minute).
* **Helmet:** Automatically sets a dozen security HTTP headers (preventing XSS, Clickjacking, etc.).
* **Strict CORS:** The API will only accept Cross-Origin Resource Sharing requests from your exact frontend domain (e.g., `https://app.gekko.ma`). Any script kiddie trying to hit the API from another website will be blocked by the browser.
* **SQL Injection Prevention:** Prisma ORM automatically uses parameterized queries under the hood, making SQL injection effectively impossible.

## 4. Protecting the Database (The "Tenant Isolation" Wall)

The biggest risk in a SaaS is the "Tenant Bleed" bug—where User A accidentally sees User B's clients or invoices. 

We **do not** rely on developers remembering to manually type `WHERE organization_id = 'xxx'` on every query. Humans make mistakes.

Instead, we protect the database using **Prisma Client Extensions (Global Scopes)**.
We configure Prisma so that every time a request comes in, Prisma intercepts it and *automatically appends* the `organization_id` constraint deep inside the ORM engine.

```typescript
// The developer writes this simple query:
const clients = await prisma.client.findMany();

// The ORM intercepts it and actually executes:
// SELECT * FROM clients WHERE organization_id = 'user_current_org_id';
```
This architectural wall guarantees that data leakage between companies is impossible, even if a junior developer writes a sloppy query.
