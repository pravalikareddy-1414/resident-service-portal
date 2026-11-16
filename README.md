# Civic — Accessible Resident Service Request Portal

A Git-ready reference project with a React/TypeScript/SASS frontend, Node.js API, PostgreSQL database, private S3 photo uploads, optional SNS SMS notifications, AWS CDK infrastructure, and automated quality checks.

## Quick start (browser demo)

Requirements: Node.js 22+ and npm.

```bash
npm ci
npm run dev
```

Open http://localhost:5173. No AWS account or database is needed. Report an issue, save its number and tracking key, and use Track a request. Demo records and photos remain in the same browser's local storage. No SMS is sent. Demo data is lost when browser storage is cleared. Use synthetic information only.

## Full local API + PostgreSQL

Requirements: Docker Compose, Node.js 22+.

```bash
cp .env.example .env
docker compose up -d --wait
node server/migrate.js
npm run dev:api
```

In a second terminal, run `npm run dev`. Vite reads `VITE_API_URL` from `.env`; restart Vite after changing it. Set `FRONTEND_ORIGIN` if using a frontend origin other than http://localhost:5173.

Local photos require an AWS S3 bucket: set `UPLOAD_BUCKET`, `AWS_REGION`, and AWS credentials through your normal local AWS profile. Configure bucket CORS to allow PUT from the frontend origin. Without S3, the local API supports requests without photos. SMS is off by default; set `SMS_ENABLED=true` only after configuring AWS SNS and obtaining SMS sending access. Never commit `.env` or credentials.

## Repository contents

| Path | Purpose |
| --- | --- |
| `src/` | Responsive React interface, API client, SASS styles |
| `server/handler.js` | Lambda handler and parameterized request queries |
| `server/schema.sql` | PostgreSQL schema and index |
| `server/local.js` | Express adapter for local development |
| `server/update-status.js` | Trusted operator status update command |
| `infra/` | CDK: API Gateway, Lambda, private RDS, S3, SNS permissions |
| `tests/` | Jest, React Testing Library, and axe tests |
| `cypress/` | Browser lifecycle and accessibility tests |
| `.github/workflows/ci.yml` | Build, tests, Cypress, and Lighthouse on PRs/pushes |

## Verification

```bash
npm test
npm run build
npm ci --prefix infra
npm run infra:build
npx cypress install
# Start npm run dev in another terminal, then:
npm run test:e2e
npm run audit:performance
```

Cypress checks the browser-demo flow. Lighthouse serves the production `dist` build and enforces performance and accessibility scores of at least 0.95 across three runs. These are configured budgets, not guaranteed measured results. Automated axe checks do not establish WCAG conformance; manually check keyboard navigation, screen readers, zoom, reflow, contrast, and error recovery before claiming WCAG 2.1 AA compliance.

Build, eight Jest tests (including axe), and infrastructure type checking passed during creation. Report submission and tracking were verified in a real browser. Cypress, Lighthouse, actual PostgreSQL integration, and AWS deployment were not executed during creation.

## AWS deployment

Requires an AWS account, configured AWS CLI credentials, Docker (Lambda bundling), Node.js 22+, and a frontend HTTPS origin. This provisions billable resources, including RDS and a NAT gateway.

```bash
npm ci
npm ci --prefix infra
cd infra
npx cdk bootstrap
npx cdk deploy -c frontendOrigin=https://YOUR-FRONTEND-DOMAIN
```

Copy the `ApiUrl` output, excluding its trailing slash, into the frontend build environment as `VITE_API_URL`. Invoke the schema migration function once after deployment:

```bash
aws lambda invoke --function-name MIGRATION_FUNCTION_OUTPUT response.json
```

Set `VITE_API_URL` in your frontend host, run `npm run build`, and host `dist/` with your preferred static host (for example S3 + CloudFront). The frontend host itself is not provisioned by the included backend stack. Origin must exactly match `frontendOrigin`; use the same value on subsequent CDK deployments.

SMS publishing code is included but disabled by default. Complete AWS SMS sandbox/region/origination registration requirements, then enable SMS in the Lambda configuration or change `SMS_ENABLED` in CDK and redeploy. Standard messaging charges apply. Initial acknowledgement is best-effort after the record is persisted.

To update a request from a trusted machine with database connectivity:

```bash
node server/update-status.js SR-REQUEST-UUID "In progress"
node server/update-status.js SR-REQUEST-UUID "Resolved"
```

The CLI uses `DATABASE_URL` locally or `DB_SECRET` as a JSON database connection object. AWS RDS is private; execute from a trusted VPC-connected environment. Status changes notify consented recipients when SNS is enabled. There is no public administrative API.

RDS deletion protection and snapshot retention are enabled. Removing the stack requires explicitly handling deletion protection and retained resources.

## API

- `POST /requests`: category, address, description, optional photo_key, phone, sms_consent (`yes`). Returns record and one-time private tracking token.
- `GET /requests/:id?token=...`: returns status and request details only when the token hash matches.
- `POST /uploads`: contentType and size; returns a short-lived S3 PUT URL and object key. Client uploads directly before creating the request.

Tracking tokens are hashed in PostgreSQL. Keep keys private. API replies disable caching; do not enable request URL/body logs that expose tokens. Photos are private and expire after 90 days. Uploaded content type/size is checked, but content malware scanning is not implemented. Production services should add abuse controls, upload ownership binding, file scanning, retention policies, operational authentication, SMS delivery retries and opt-out handling, and an agency-specific privacy notice. Requests are categorized for review; there is no agency dispatch integration.

## Push to your Git repository

Unzip the folder, create an empty remote repository, then run these commands inside `resident-service-portal`:

```bash
git init
git add .
git commit -m "Build accessible resident service portal"
git branch -M main
git remote add origin https://github.com/YOUR-USERNAME/YOUR-REPOSITORY.git
git push -u origin main
```

Both package lockfiles are included. Dependencies, generated builds, credentials, and local environment files are excluded by `.gitignore`. This project is a demonstration and is not affiliated with a municipality.
