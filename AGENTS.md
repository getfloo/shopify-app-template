# Shopify app on floo

An embedded Shopify app built on Shopify's React Router app template, deployed on
floo. Shopify's docs for that template apply: https://shopify.dev/docs/api/shopify-app-react-router

## floo-specific rules
- Keep `access_mode = "public"` in `floo.app.toml`. Shopify authenticates every
  request (session tokens in the admin, HMAC on webhooks) through `authenticate.*`
  in `app/shopify.server.ts`; floo sign-in in front would block Shopify.
- Keep `addDocumentResponseHeaders` in `app/entry.server.tsx`. It sends the
  `frame-ancestors` header that lets Shopify admin embed the app; floo's gateway
  sends `X-Frame-Options: SAMEORIGIN`, which that header overrides.
- Data lives in floo's managed Postgres through Prisma (`prisma/schema.prisma`,
  `DATABASE_URL`). Add a table by editing the schema and adding a migration under
  `prisma/migrations/`; floo runs `prisma migrate deploy` once per deploy.
- Secrets go in floo env vars (`floo env set KEY --stdin`), never in the repo.
  `SHOPIFY_API_KEY`, `SHOPIFY_API_SECRET` and `SCOPES` are required, and `SCOPES`
  must match `scopes` in `shopify.app.toml`.
- Webhook routes must answer quickly. Shopify retries a delivery that takes longer
  than five seconds, so move slow work to a floo cron job or worker service.
- Declare webhook subscriptions in `shopify.app.toml` and release them with
  `shopify app deploy`; the route files under `app/routes/webhooks.*` handle them.
