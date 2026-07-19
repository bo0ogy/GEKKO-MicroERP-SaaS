# GEKKO-Entreprise: Folder & Code Structure (NestJS)

Because we chose **NestJS** (Modular Monolith), the file structure is strictly dictated by **Domain-Driven Design (DDD)**. NestJS forces code organization by "Feature" (or Module) rather than throwing all controllers into one massive `controllers/` folder.

This structure handles the complexity of an ERP system by keeping domains strictly isolated.

## 📁 The Folder Structure

```text
src/
├── main.ts                     # Entry point of the application
├── app.module.ts               # The root module that imports all other modules
├── prisma/                     # Database layer
│   └── schema.prisma           # All MODELS live here (Organization, User, Invoice, etc.)
│
├── common/                     # Global utilities used across the app
│   ├── middlewares/            # e.g., TenantIsolationMiddleware
│   ├── guards/                 # e.g., JwtAuthGuard, RolesGuard
│   ├── decorators/             # e.g., @CurrentUser(), @RequirePermissions()
│   └── filters/                # Global Error Handling
│
└── modules/                    # The heart of the application
    ├── crm/                    # (Sprint 2)
    ├── quotes/                 # (Sprint 3)
    └── invoicing/              # (Sprint 4)
        ├── invoicing.module.ts # Wires the invoicing domain together
        ├── controllers/        
        │   └── invoice.controller.ts  # Routes handling (GET, POST, etc.)
        ├── services/           
        │   ├── invoice.service.ts     # BUSINESS LOGIC (Core operations)
        │   └── invoice-pdf.service.ts # Specific logic (e.g., PDF Generation)
        ├── dtos/               
        │   ├── create-invoice.dto.ts  # VALIDATIONS (Input shape and rules)
        │   └── update-invoice.dto.ts
        └── events/             
            └── invoice.listener.ts    # Listens to internal events (e.g., QuoteAccepted)
```

## 🔍 Where does everything live?

### 1. Models (Database Schema)
All models live centrally in `prisma/schema.prisma`. We don't use scattered model files. Prisma acts as the Single Source of Truth. It auto-generates perfect TypeScript interfaces so every part of the app knows exactly what an `Invoice` or a `Client` looks like.

### 2. Routes & Controllers (`controllers/`)
Controllers are *only* responsible for HTTP traffic. An `invoice.controller.ts` will define `@Get()`, `@Post()`, parse the incoming request, call the Service, and return the HTTP response. **There is ZERO business logic in a controller.**

### 3. Validations (`dtos/`)
Validations live in **DTOs (Data Transfer Objects)**. 
A `create-invoice.dto.ts` uses decorators (like `@IsString()`, `@IsUUID()`, `@Min(0)`). When a request hits the server, NestJS automatically intercepts it, runs the validation rules in the DTO, and instantly rejects it with a `400 Bad Request` if it fails—before it even reaches your controller.

### 4. Business Logic (`services/`) [🚨 VERY IMPORTANT]
*All* business logic lives in the `services/` directory. 
If you need to convert a Quote to an Invoice, calculate the TVA, verify that the `discount_amount` doesn't exceed the subtotal, or ensure chronological invoice numbers, **it happens in the Service**. Services are highly testable classes that don't know anything about HTTP.

### 5. Middlewares & Guards (`common/middlewares/` and `common/guards/`)
* **Middlewares:** Used for things like request logging or intercepting the `organization_id` from the HTTP Headers.
* **Guards:** Used for Security. A `RolesGuard` will sit above a controller route and check: *"Does the user in the current JWT have the 'Comptable' role?"* If not, it blocks the request.

## 🛡️ Why won't this become messy after 3 months?

1. **Domain Isolation:** In traditional frameworks (like Express), developers group files by *Type* (one giant folder with 50 controllers, another with 50 services). In this structure, we group by **Domain** (the `invoicing/` folder contains everything related to invoicing). If you have a bug in invoices, you know exactly which folder to look in. It limits the "cognitive load" as the ERP grows.
2. **Dependency Injection:** NestJS uses Dependency Injection heavily. If the `InvoicingService` needs to fetch a Client, it doesn't query the database directly. It asks the `CrmService` for the client. This prevents tightly coupled "spaghetti code."
3. **Strict Boundaries for Business Logic:** Because we enforce the rule *"No logic in controllers"*, your business logic remains pure. You can test your `InvoiceService.calculateTaxes()` function in milliseconds without having to spin up fake HTTP requests or servers.
4. **Enforced Consistency:** NestJS has an opinionated structure. If you hire a new developer in month 4, they can't randomly decide to write a new route using `app.get('/path')`. They are forced to use the Module/Controller/Service pattern, keeping the codebase completely uniform.
