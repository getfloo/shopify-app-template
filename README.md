# Shopify app template for floo

An embedded Shopify app that runs on floo: it opens inside Shopify admin, signs merchants in with Shopify, stores their sessions in floo's managed Postgres, and receives webhooks. It is Shopify's [React Router app template](https://github.com/Shopify/shopify-app-template-react-router) with the changes floo needs, so Shopify's docs for that template apply here.

## What changed from Shopify's template

- **Postgres instead of SQLite.** `prisma/schema.prisma` reads `DATABASE_URL`, which floo injects from `[managed.default]` in `floo.app.toml`. The session table migration is rewritten for Postgres.
- **Migrations run once per deploy.** `migrate_command` runs `prisma migrate deploy` before traffic moves; a failed migration stops the deploy and the previous revision keeps serving. The container only starts the server.
- **The app URL comes from floo.** On floo, `app/shopify.server.ts` uses `FLOO_APP_URL`, the app's public URL. `shopify app dev` still sets `SHOPIFY_APP_URL` to its tunnel locally.
- **Public access mode.** Shopify verifies every request itself: the embedded admin sends a session token and webhooks carry an HMAC signature. floo's sign-in can't sit in front of either, so `access_mode = "public"`.
- **A committed `package-lock.json` and `shopify.web.toml`,** so `npm ci` and `shopify app dev` work from a clone.

The app sends `Content-Security-Policy: frame-ancestors` for the merchant's shop and `admin.shopify.com` on every page (Shopify's `addDocumentResponseHeaders` in `app/entry.server.tsx`). Keep it: floo's gateway sends `X-Frame-Options: SAMEORIGIN`, and browsers let the app's `frame-ancestors` override it. Without that header, the app shows a blank frame inside Shopify admin.

## Deploy it

You need a Shopify Partner account, a development store, and the [Shopify CLI](https://shopify.dev/docs/apps/tools/cli).

1. Click **Use this template** on GitHub to create your own repository, and clone it. Change `name` in `floo.app.toml` if you want a different app name.
2. Create the Shopify app and link this folder to it:

   ```sh
   npm install
   shopify app config link
   ```

3. Deploy from the folder. floo connects your GitHub repository, creates the app and starts a deploy, which stops and lists the three variables it needs:

   ```sh
   floo deploy
   ```

4. Copy the client ID and secret from your app in the Shopify Dev Dashboard and set them on floo. `SCOPES` must match `scopes` in `shopify.app.toml`:

   ```sh
   floo env set SHOPIFY_API_KEY=<client id>
   floo env set SHOPIFY_API_SECRET --stdin
   floo env set SCOPES=write_products
   floo deploy --rebuild
   ```

5. Point Shopify at the floo URL. In `shopify.app.toml`, set `application_url` to your app's URL (`https://<name>.getfloo.app`) and `redirect_urls` under `[auth]` to `https://<name>.getfloo.app/auth/callback`, then release the configuration:

   ```sh
   shopify app deploy
   ```

6. Install the app on your development store from the Dev Dashboard and open it in Shopify admin.

The full walkthrough, including production stores and local development, is at [getfloo.com/docs/deploy/shopify-app](https://getfloo.com/docs/deploy/shopify-app).

## Customize

Give your agent prompts like these:

- "Add a page that lists the store's 20 most recent orders with totals, using the Admin GraphQL API."
- "Subscribe to the `orders/create` webhook and save each order's ID, total and created time in a new Postgres table."
- "Add a daily floo cron job that records products whose inventory is below 5."
- "Add the three mandatory compliance webhooks for App Store review and delete a shop's data on `shop/redact`."

## Local development

`shopify app dev` runs the app behind a Shopify tunnel and needs a `DATABASE_URL`. `floo run` injects your floo dev database's credentials into a command:

```sh
floo run --service web -- shopify app dev
```

If `shopify app dev` updates your app's URLs to its tunnel, run `shopify app deploy` afterwards to point them back at floo.

## License

MIT. Based on Shopify's app template, copyright Shopify Inc.; see `LICENSE.md`.
