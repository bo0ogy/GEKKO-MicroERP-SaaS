# GEKKO-Entreprise: Environment & Configuration

Handling environment variables and deployment strategies correctly from day one prevents catastrophic security leaks and late-night server crashes.

## 1. Managing Environment Variables & Secrets

### A. The Fail-Fast Strategy
We do not just load a `.env` file and hope for the best. We use `@nestjs/config` combined with `class-validator` to strictly validate our environment variables at startup.
* If a developer starts the server but forgets to set `JWT_SECRET` or `DATABASE_URL`, the application will instantly crash with a clear error: `Error: JWT_SECRET is required`.
* This "fail-fast" approach prevents the ERP from running in a semi-broken state.

### B. Security
* We maintain a `.env.example` in the repository with fake values so new developers know what is required.
* The real `.env` file is strictly listed in `.gitignore` and never committed.
* In production, we don't use `.env` files at all. Secrets are injected directly into the server environment (e.g., via GitHub Actions Secrets, AWS Secrets Manager, or DigitalOcean Environment Variables).

## 2. How the App Runs (Dev vs. Prod)

The environment dictates how the application is executed to maximize developer speed locally and maximize raw performance globally.

### A. Development Environment
* **Execution:** We run the app using `npm run start:dev`. This uses `ts-node-dev` to watch for file changes and instantly restarts the server (Hot Reloading) when you save a file.
* **Database:** The local PostgreSQL database (and Redis queue) runs inside a lightweight Docker container via `docker-compose.yml`. You don't need to install Postgres on your laptop.
* **Migrations:** We use `npx prisma migrate dev`, which actively watches the `schema.prisma` file, generates the SQL history, and updates the local database.

### B. Production Environment
* **Execution:** We **never** run TypeScript in production because it is too slow and memory-heavy. The CI/CD pipeline runs `npm run build`, which strips out all types and compiles highly optimized pure JavaScript into a `dist/` folder. The server is then started simply with `node dist/main.js` (or managed by PM2/Docker).
* **Dockerized:** The entire production app is built into a standalone Docker image via a `Dockerfile`. This ensures that "it works on my machine" translates to "it works flawlessly on the production server."
* **Safe Migrations:** In production, we do not use `migrate dev`. The deployment pipeline runs `npx prisma migrate deploy`. This command does not modify the schema; it simply applies the exact, reviewed SQL migration files from the `prisma/migrations` folder safely to the live database before the new code boots up.
