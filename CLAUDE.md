# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Package manager is **pnpm**; Node `^22.19.0 || ^24.11.0 || >=26.0.0`.

```bash
pnpm dev            # nuxt prepare && nuxt dev -> http://localhost:3000
pnpm build          # production build; pnpm preview serves .output
pnpm lint           # oxlint . && oxfmt --check .   (no config files -> tool defaults)
pnpm lint:fix       # oxlint --fix, oxfmt --write, then typecheck
pnpm typecheck      # nuxt prepare && nuxi typecheck (vue-tsc, strict)
pnpm generate-types # shopware-api-gen generate --apiType=store
```

There is no test runner and no test suite in this repository. "Verifying a change" means
`pnpm typecheck` plus `pnpm lint`, and running `pnpm dev` when behavior matters.

Anything that touches `.nuxt` generated types (new composable, new component dir, config change)
needs `nuxt prepare` first — every script above already does it.

### API types

`shopware.d.ts` declares the `#shopware` module (`operations`, `Schemas`, `ApiClient`) and currently
points at the types shipped with `@shopware/api-client`. To use a real instance's schema, run
`npx shopware-api-gen loadSchema --apiType=store` **before** `pnpm generate-types` (otherwise the
generator falls back to the published defaults), then switch the imports in `shopware.d.ts` to
`./api-types/storeApiTypes`.

## Configuration

`nuxt.config.ts` holds `runtimeConfig` defaults for the Shopware connection; `.env` overrides them
via `NUXT_PUBLIC_SHOPWARE_ENDPOINT` / `NUXT_PUBLIC_SHOPWARE_ACCESS_TOKEN` (see `.env.template`).
The committed defaults in `nuxt.config.ts` currently differ from the demo shop documented in the
README and `.env.template` — read the config, don't assume the demo endpoint.

`cacheableReads: true` routes anonymous Store API reads through cacheable GET endpoints; a backend
without the GET read routes / `_criteria` param needs it set to `false`.

## Architecture

### Nuxt layers

The app extends three Shopware layers, and most behavior comes from them rather than from this
repository:

- `@shopware/composables/nuxt-layer` — `useCart`, `useUser`, `useSessionContext`, `useProductSearch`,
  `useNavigation`, `useWishlist`, `useInternationalization`, … (all auto-imported)
- `@shopware/cms-base-layer` — Shopping Experiences rendering: `CmsPage`, `CmsGenericBlock`,
  `CmsGenericElement`, `CmsBlock*`, `CmsElement*`, `Sw*` components, plus the `@nuxt/image`
  `shopware` provider and presets
- `@shopware/unocss-design-tokens-layer` — UnoCSS presets and design tokens; `uno.config.ts` merges
  template-specific safelists/fonts on top with `mergeConfigs`

### Page resolution

There are almost no catalog routes in `app/pages`. `app/pages/[...all].vue` is the resolver:

1. strips the i18n locale prefix from the path, calls `resolvePath` (Store API SEO lookup),
2. redirects technical paths to their canonical SEO path (301), 404s when nothing resolves,
3. turns the resolved `routeName` into a PascalCase component name and `resolveComponent`s it.

Those targets live in `app/components/global/` (`FrontendDetailPage`, `FrontendLandingPage`,
`FrontendNavigationPage`) and each fetches its entity, calls `useCmsHead`, and renders
`<CmsPage :content="…cmsPage" />`. Explicit pages exist only for account, checkout, cart, wishlist,
search, newsletter and registration confirmation.

### Component auto-import groups

`nuxt.config.ts` defines three component groups deliberately; read the comments there before
changing them:

- `app/components/global/` — `global: true`, `pathPrefix: false`. Required because `[...all].vue`
  uses `resolveComponent`.
- `app/components/cms/` — `global: true`, `pathPrefix: false`. Drop a file named exactly like a
  layer component (`CmsBlockImageText.vue`, `CmsElementText.vue`, `SwProductCard.vue`) here to
  override it. Currently only a `.gitkeep`.
- `app/components/` — normal path-prefixed auto-import: `layout/Header.vue` → `<LayoutHeader>`,
  `account/menu/list.vue` → `<AccountMenuList>`, `shared/Modal.vue` → `<SharedModal>`,
  `form/InputField.vue` → `<FormInputField>`.

Each global group needs its own directory path — Nuxt skips later scans under an already-scanned
path — which is why the two global dirs come first.

Extra auto-import dirs (`imports.dirs`): `app/utils`, `app/utils/**`, `i18n/utils`,
`i18n/src/helpers`.

### i18n

Two directory trees, both used:

- `i18n/<locale>/*.json` — the actual messages, split by domain (`account`, `checkout`, `errors`,
  `validations`, …) and spread into one object by `i18n/<locale>/<locale>.ts`.
- `i18n/src/langs/<locale>.ts` — the files `langDir` points at; each just re-exports the above.

Strategy is `prefix_except_default` with `en-GB` as default. `app/app.vue` derives the prefix from
the route name (`getPrefix`), sets `locale`, and — when it disagrees with the session — applies
`sw-language-id` to the API client and calls `changeLanguage` + `refreshSessionContext`. It also
`provide`s `cmsTranslations` and `urlPrefix` for the CMS layer. Internal links must go through
`useLocalePath()` / `useInternationalization().formatLink`, not raw paths.

Adding a message means editing the JSON for every locale; the keys are English, the values are the
translations.

### Forms and validation

`@regle/core` + `@regle/rules`, wrapped by `customValidators()` in `i18n/utils/i18n-validators.ts`,
which attaches `validations.*` i18n messages via `withMessage`. Per-form rule factories live in
`app/utils/validation/rules/*.ts` and are auto-imported.

### API errors

`useApiErrorsResolver(context?)` maps `ApiClientError` details to `errors.<code>` translations,
falls back to the raw `detail`, and pushes through `useNotifications`. Its `contextErrors` map
patches Store API error codes that are ambiguous per context (e.g. `account_login`). Use it rather
than surfacing raw API errors.

### App-level state

`app/app.vue` is where the session bootstraps: it reads `/context`, loads languages, hydrates the
wishlist ids (capped at 100 — the header counter uses `total-count-mode: exact` and is unaffected),
refreshes the cart on mount, and provides the login modal via `provideLoginModal()` /
`useLoginModal()`. Shared modal controllers use `createSharedComposable(useModal)`
(`useSideMenuModal`, `useMiniCartModal`). `useAuthGuardRedirection()` is called from
`app/layouts/account.vue` and is client-side only.

### Caching, SSR and route rules

`routeRules` in `nuxt.config.ts` sets 24h ISR plus `Surrogate-Control` for `/**`, and turns SSR
**off** with `no-store` for `/checkout`, `/account`, `/wishlist` and their subtrees. Anything
session- or currency-dependent rendered on a shared-cached route will leak or flicker: if you enable
`useUserContextInSSR`, the shared HTML cache on `/**` has to go.

### Server routes

Nitro routes live in `server/routes/`. `account/login/imitate-customer.post.ts` creates its own API
client from runtime config, exchanges the admin imitation token, and sets the `sw-context-token`
cookie before redirecting to `/account`.
