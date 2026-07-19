# GEKKO-Entreprise: API Design

## 1. REST vs GraphQL

**We will use strict REST API architecture.**

While GraphQL is powerful for highly interconnected frontend graphs (like social networks), **REST is the absolute industry standard for ERPs and B2B SaaS.** 
* **Why REST?** It is highly predictable, easier to cache, and extremely simple to secure with rate-limiting. Furthermore, if GEKKO ever needs to expose a public API to third-party tools (e.g., connecting a user's webshop to the ERP, or connecting to Moroccan tax endpoints), REST is universally understood and expected.

## 2. Real Endpoint Examples

Our API will be prefixed with versioning (`/api/v1/`) to ensure we don't break the frontend when making large updates in the future.

### A. List Invoices
**Request:** `GET /api/v1/invoices?status=Payée&page=1&limit=20`
* **Purpose:** Fetches a paginated list of invoices. The `organization_id` is automatically extracted from the user's JWT, so they can never see another tenant's invoices.

### B. Create Invoice
**Request:** `POST /api/v1/invoices`
**Payload:**
```json
{
  "client_id": "uuid-of-client",
  "items": [
    { "product_id": "uuid-product-1", "quantite": 5, "discount_amount": 0 },
    { "product_id": "uuid-product-2", "quantite": 1, "discount_amount": 10 }
  ]
}
```
* **Purpose:** Triggers the complex transaction logic. Notice there are no prices sent.

### C. Update Invoice Status
**Request:** `PATCH /api/v1/invoices/:id/status`
**Payload:**
```json
{
  "status": "Payée",
  "payment_method": "Virement"
}
```
* **Purpose:** We use `PATCH` (not PUT) because we are only modifying a specific subset of the resource, not replacing the entire invoice.

## 3. Validation, Errors, and Consistency

The biggest problem with messy APIs is that Route A returns an array, Route B returns an object, and Route C returns a string when an error occurs. We solve this elegantly using NestJS core features.

### A. Validation (DTOs)
We use `class-validator` via NestJS Pipes. 
When the frontend sends the `POST /api/v1/invoices` payload, NestJS runs it through a `CreateInvoiceDto` class.
* If the `client_id` is missing, or `quantite` is `-5` instead of a positive number, NestJS **automatically blocks the request** and returns a `400 Bad Request` with an exact array of what went wrong.
* The code inside your Controller *does not even execute* if the validation fails.

### B. Response Format Consistency (Interceptors)
We will create a Global Interceptor. This guarantees that **every successful response** in the entire application follows the exact same shape:
```json
{
  "success": true,
  "data": {
     // ... the actual invoice or list ...
  },
  "meta": {
    "timestamp": "2026-04-28T18:00:00Z",
    "path": "/api/v1/invoices"
  }
}
```

### C. Error Handling (Global Filters)
We never use `try/catch` inside every single controller. Instead, we use a **Global Exception Filter**. 
If a service throws a `NotFoundException` or if the database crashes, the Exception Filter catches it at the top level and formats it perfectly:
```json
{
  "success": false,
  "error": {
    "code": 404,
    "message": "Invoice FA-2024-001 not found",
    "details": null
  },
  "meta": {
    "timestamp": "2026-04-28T18:01:00Z",
    "path": "/api/v1/invoices/FA-2024-001"
  }
}
```
This means your frontend developer only needs to write **one** Axios interceptor to handle errors for the entire application, saving countless hours of debugging.
