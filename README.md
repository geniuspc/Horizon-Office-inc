# Horizon

## My Contribution (Backend Architecture & DevOps)
As a Backend Developer on this project, I was responsible for the Go microservice and the infrastructure deployment. My key technical contributions include:
- **Go Microservice (`go-api`):** Designed and developed a secure authentication REST API using **Go** and the `chi` router.
- **Authentication:** Implemented custom **JWT-based auth** (login/registration) with bcrypt hashing and integrated **Google SSO (OAuth2)**.
- **Database Integration:** Configured high-performance database connection pooling to **PostgreSQL** using `pgx`.
- **DevOps & Cloud (AWS):** Containerized backend services with **Docker/Docker Compose** and deployed them to **AWS EC2** instances within a VPC, including network and environment configuration for production.

---

Volunteer quests in Dnipro. Organizations publish good deeds on a 3D city map, volunteers sign up, gain experience and coins, and exchange them for partner rewards.

## Structure

| Folder | What is it | Stack | Port |
|---|---|---|---|
| `web/` | Website | Next.js 16, React 19, Tailwind 4, three.js | 3001 |
| `express-api/` | Quests, applications, organizations, profile, rewards | Express 5, Prisma 7, PostgreSQL | 3000 |
| `go-api/` | Login and registration: password or Google, issues JWT | Go, chi, pgx | 8080 |

One PostgreSQL database (AWS RDS) for both APIs. The schema is owned by `express-api` (Prisma migrations).

## How it works

- **Login.** `go-api` issues a JWT. Next places it in an httpOnly-cookie, `web/proxy.ts` verifies the session before page rendering and passes the token to `express-api`.
- **Volunteer.** Quest map, applying and canceling applications, "My quests", profile with level, reward shop, and a gift for every 5 completed quests.
- **Organization.** The dashboard unlocks after completing the organization profile: quests, volunteer applications, confirming completion with XP and coin rewards.

## Running Locally

Requires Node.js 20.19+, Go 1.26+, and an SSH tunnel to the database at `localhost:5433` (see `express-api/docs/DB_TUNNEL.md`).

```bash
# 1. express-api
cd express-api && npm install && npm run migrate:deploy && npm run db:seed && npm run dev

# 2. go-api
cd go-api && go run ./cmd/api

# 3. web
cd web && npm install && npm run dev
```

Website: http://localhost:3001 · API Documentation: http://localhost:3000/docs

## Environment Variables

Each service reads its own `.env` (does not get committed to git; examples are in `.env.example`).

| Variable | Where | What |
|---|---|---|
| `DATABASE_URL` | express-api, go-api | Postgres via tunnel |
| `JWT_SECRET`, `JWT_ISSUER`, `JWT_AUDIENCE` | all three | Identical everywhere: go-api signs, the rest verify |
| `CLIENT_ID` / `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | go-api / web | Client ID from Google Cloud Console |
| `NEXT_PUBLIC_API_URL=/api`, `AUTH_API_URL` | web | API addresses for the Next proxy |

## Deployment (AWS)

Two EC2 instances in one VPC, Docker Compose, HTTPS provided by CloudFront (`https://d203ezrrdmg2gd.cloudfront.net` → frontend :80).

| Instance | What | Command (from repository root) |
|---|---|---|
| backend | express-api :3000, go-api :8080 | `docker compose -f docker-compose.backend.yml up -d --build` |
| frontend | web :80 | `docker compose -f docker-compose.frontend.yml --env-file web/.env up -d --build` |

`express-api` applies migrations automatically on startup. Reward shop seed: `docker compose -f docker-compose.backend.yml exec express-api npm run db:seed`.

Differences between production and local `.env`:

- `express-api/.env`, `go-api/.env`: `DATABASE_URL` points directly to the RDS endpoint (no tunnel), `IS_DEVELOPMENT=false`, no `DEV_BYPASS_TOKEN`.
- `web/.env`: `API_URL=http://<backend private IP>:3000`, `AUTH_API_URL=http://<backend private IP>:8080`, no `NEXT_PUBLIC_API_TOKEN`. `API_URL` and `NEXT_PUBLIC_*` are baked in during build: rebuild the image after changes.
- `JWT_*` are identical on both instances. The CloudFront domain must be in Google Console → Authorized JavaScript origins.

## Useful

- `npm run db:seed` in `express-api` — dev user and reward shop. Can be run repeatedly, no duplicates.
- `npm test` in `express-api` — autotests. They recreate the schema, so `.env.test` must point to a separate database.