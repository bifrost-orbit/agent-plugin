---
name: cms-orbit-build-frontend
description: Use when writing website or app code that reads content from CMS Orbit with the JavaScript SDK @bifrost-orbit-cms/sdk. Covers the delivery client, rendering pages and blocks, generated types, images, links, rich text, regions and languages, preview and draft mode, live updates in the visual editor, and refreshing caches with webhooks, for Next.js (App Router) and Astro. For example "set up CMS Orbit in this Next.js project", "write the component for the trip hero block", "make the preview work in the editor".
---

# Build a website on CMS Orbit with the SDK

`@bifrost-orbit-cms/sdk` is the JavaScript SDK: a typed client for the delivery API, a React renderer, Next.js helpers for preview, cache refresh and sitemaps, SEO output and a type generator. It needs Node.js 20 or later. React 19 and Next.js 16 are optional peers.

| Import | For |
| --- | --- |
| `@bifrost-orbit-cms/sdk` | `createCmsClient`, errors, `cacheTags`, image helpers, `verifyWebhook`, `tagsForEvent` |
| `@bifrost-orbit-cms/sdk/react` | `CmsPage`, `CmsLivePage`, `useLivePage`, `CmsBlocks`, `defineRegistry`, `BlockProps`, `CmsImage`, `RichText` |
| `@bifrost-orbit-cms/sdk/next` | `createPreviewHandler`, `createPreviewExitHandler`, `createRevalidateHandler`, `CmsBridge`, `followResolution`, `gone` |
| `@bifrost-orbit-cms/sdk/seo` | `toMetadata`, `hreflang`, `JsonLd`, sitemap XML |
| `@bifrost-orbit-cms/sdk/testing` | `createMockClient`, `fixtures` |

For Astro, read `references/astro.md` after this file. For other frameworks (React, Nuxt, Vue, SvelteKit, Angular, Laravel, Symfony, Swift, Kotlin), follow the matching quickstart at https://docs.bifrostorbit.com/cms/developers/quickstarts/.

## Steps, in order (Next.js App Router)

A new project can start from a starter that is already connected: `npx cms-orbit project create northwilds-site --framework nextjs --create-key` (see `cms-orbit-use-cli`). Otherwise:

1. **Install:** `npm install @bifrost-orbit-cms/sdk server-only`.
2. **Settings** in `.env.local`, which Next.js keeps out of git. Never give them a `NEXT_PUBLIC_` prefix:
   ```bash
   CMS_API_URL=https://api.bifrostorbit.com/cms
   CMS_PUBLIC_KEY=cms_pk_...    # published content
   CMS_PREVIEW_KEY=cms_sk_...   # drafts, server only, with Read drafts on
   CMS_DEFAULT_REGION=global    # the region's key, from Configuration › Environments
   CMS_WEBHOOK_SECRET=          # whsec_..., from Configuration › Webhooks
   ```
   The person creates the keys under **Configuration › API keys**. Do not ask them to paste a secret key into the chat.
3. **The client**, in one server-only file:
   ```ts
   // lib/cms.ts
   import 'server-only'
   import { createCmsClient } from '@bifrost-orbit-cms/sdk'

   export const cms = createCmsClient({
     baseUrl: process.env.CMS_API_URL!,
     apiKey: process.env.CMS_PUBLIC_KEY!,
     previewKey: process.env.CMS_PREVIEW_KEY || undefined,
     defaultRegion: process.env.CMS_DEFAULT_REGION,
     cache: { revalidate: 60, tags: true },
   })
   ```
4. **One catch-all route** renders every page. `cms.resolve` answers `page`, `redirect`, `gone` or `notFound`:
   ```tsx
   // app/[[...slug]]/page.tsx
   import type { Metadata } from 'next'
   import { draftMode } from 'next/headers'
   import { notFound, permanentRedirect, redirect } from 'next/navigation'
   import { cache } from 'react'
   import { gone } from '@bifrost-orbit-cms/sdk/next'
   import { toMetadata } from '@bifrost-orbit-cms/sdk/seo'

   import { PageBody } from '@/components/page-body'
   import { cms } from '@/lib/cms'

   type Props = { params: Promise<{ slug?: string[] }> }

   export const revalidate = 60

   export function generateStaticParams() {
     return []
   }

   const load = cache(async (path: string, preview: boolean) => cms.resolve({ path, preview }))

   async function resolveFor({ params }: Props) {
     const { slug = [] } = await params
     const { isEnabled } = await draftMode()
     return load(`/${slug.join('/')}`, isEnabled)
   }

   export async function generateMetadata(props: Props): Promise<Metadata> {
     const result = await resolveFor(props)
     return result.kind === 'page' ? toMetadata(result.page) : {}
   }

   export default async function Page(props: Props) {
     const result = await resolveFor(props)
     if (result.kind === 'redirect') {
       if (result.permanent) permanentRedirect(result.to)
       redirect(result.to)
     }
     if (result.kind === 'gone') gone()
     if (result.kind === 'notFound') notFound()
     return <main><PageBody page={result.page} /></main>
   }
   ```
   New Next.js 16 apps turn on Cache Components by default, which makes `export const revalidate` fail the build. Create the app with `--no-cache-components`, or leave it off in `next.config.ts`.
5. **Generate types** from the space, and run it again whenever the schema changes. Add to `package.json`:
   ```json
   { "scripts": { "cms:types": "cms-orbit generate --out cms/types.ts" } }
   ```
   It reads `CMS_API_URL` and `CMS_PUBLIC_KEY` from `.env.local`. When the schema is kept in files, `npx cms-orbit types generate --from cms/schema --out cms/types.ts` reads the files instead. Run it in CI, so a schema change that breaks the website fails the build.
6. **Write a component per block** and register it. Read the block's fields in `cms/types.ts`, or with the MCP tool `get_schema`. Use the space's own keys:
   ```tsx
   // components/blocks/trip-hero.tsx
   import { CmsImage, type BlockProps } from '@bifrost-orbit-cms/sdk/react'

   export function TripHero({ block }: BlockProps<'trip-hero'>) {
     return (
       <section data-cms-id={block.id} className={block.season ?? undefined}>
         <h1 data-cms-field="heading">{block.heading}</h1>
         <CmsImage image={block.photo} sizes="100vw" priority />
       </section>
     )
   }
   ```
   A block with slots renders them from `slots`, already drawn with the registry:
   ```tsx
   // components/blocks/gear-list.tsx
   import type { BlockProps } from '@bifrost-orbit-cms/sdk/react'

   export function GearList({ block, slots }: BlockProps<'gearList'>) {
     return (
       <section data-cms-id={block.id}>
         <h2>{block.heading}</h2>
         <ul>{slots.items}</ul>
       </section>
     )
   }
   ```
   ```tsx
   // cms/registry.tsx
   import '@/cms/types'
   import { defineRegistry } from '@bifrost-orbit-cms/sdk/react'
   import { GearItem } from '@/components/blocks/gear-item'
   import { GearList } from '@/components/blocks/gear-list'
   import { TripHero } from '@/components/blocks/trip-hero'

   export const registry = defineRegistry({ 'trip-hero': TripHero, gearList: GearList, gearItem: GearItem })
   ```
   With `defineRegistry`, a component missing from the registry is a type error. While you add components one at a time, type it as `CmsRegistry` instead. An unknown block renders a dashed note in development and nothing in production.
7. **Preview and live updates**, below.
8. **Refresh on publish**, below.

## Blocks and fields

- `BlockProps<'key'>` gives `block`, with the fields typed from `cms/types.ts`, and `slots`, the nested blocks of each `component` or `components` field already rendered with the registry. `block.<slot>` is the raw node (`{ id, type, fields, children, variant }`), or a list for `components`.
- Put `data-cms-id={block.id}` on every component's outer element. Editors click it in the preview to select the block; without it the parent is selected. `data-cms-field="<field key>"` lets the editor open that field from an accessibility issue.
- **Rich text** is an HTML string. Render it with `<RichText value={…} />`. The SDK does not sanitise it; sanitise on the server, for example with `sanitize-html`, if you need a guarantee.
- **Images** arrive as `{ id, url, mimeType, width, height, alt, title, focalPoint }` or `null`. `<CmsImage image={…} sizes="(min-width: 768px) 50vw, 100vw" />` adds a `srcset` with AVIF first, width and height, lazy loading, and the editor's focus point. Add `priority` for the main image near the top. For your own markup: `cmsImageUrl(url, { width: 800 })`, `cmsImageSrcSet(url, width)`, `cmsFocalPosition(image)`. With `next/image`, allow the image host in `images.remotePatterns`.
- **Links** arrive as `{ kind, href, label, target, contentId, assetId }` or `null`. `href` is the page's current address, or `null` when the page it points at is not published: render the text without a link, or nothing. Use `next/link` when `href` starts with `/`.
- **Select** fields are one of the option values; **references** are ids unless you read with `include: ['references']` and generate types with `--include references`.
- Optional fields can be missing or `null`. The generated types mark them with `?`.

## Regions and languages

Every call takes `region` and `locale`. `region` defaults to the client's `defaultRegion`, `locale` to the region's default language.

```ts
await cms.resolve({ path: '/trips', region: 'germany', locale: 'en-DE' })
```

- Pass the region's **key**, not its address prefix. Cache tags and webhooks use the key, so a page cached under another name is never refreshed.
- The path is the same in every region: `/trips`, not `/de/trips`. Take the prefix off in your route (`app/[region]/[[...slug]]/page.tsx`) before calling.
- `page.meta` says which region and language answered and whether a fallback was used. `toMetadata` writes hreflang alternates.

Address patterns, domains per region and language switchers: https://docs.bifrostorbit.com/cms/developers/regions-and-languages/

## Preview and draft mode

The editor opens the website at the space's **Preview URL** plus the page's address and `?preview=<token>`. A token is for one page's draft and lasts an hour.

```ts
// proxy.ts (Next.js 16's name for middleware.ts), next to app/
import { NextResponse, type NextRequest } from 'next/server'

export function proxy(request: NextRequest) {
  const token = request.nextUrl.searchParams.get('preview')
  if (!token) return NextResponse.next()
  const target = new URL('/api/preview', request.url)
  target.searchParams.set('preview', token)
  target.searchParams.set('path', request.nextUrl.pathname)
  return NextResponse.redirect(target)
}

export const config = {
  matcher: [{ source: '/((?!api/|_next/).*)', has: [{ type: 'query', key: 'preview' }] }],
}
```

```ts
// app/api/preview/route.ts
import { createPreviewHandler } from '@bifrost-orbit-cms/sdk/next'
import { cms } from '@/lib/cms'
export const GET = createPreviewHandler({ client: cms })
```

```ts
// app/api/preview/exit/route.ts
import { createPreviewExitHandler } from '@bifrost-orbit-cms/sdk/next'
export const GET = createPreviewExitHandler()
```

Add `<CmsBridge client={cms} />` from `@bifrost-orbit-cms/sdk/next` inside `<body>` in `app/layout.tsx`. It renders only in draft mode, so visitors never load it. Mark the header, footer and banner with `data-cms-edit="global:header navigation:header"` and `data-cms-edit-label="Header"` so editors can open where they are edited.

Then the person sets **Preview URL** in the space's **Settings**. The website must be on `https` and allow `https://app.bifrostorbit.com` to frame it.

## Live updates as editors type

With the bridge alone the preview reloads after each save. To redraw as editors type, render drafts with `CmsLivePage` in a client component:

```tsx
// components/live-page.tsx
'use client'
import type { CmsPage as CmsPageData } from '@bifrost-orbit-cms/sdk'
import { CmsLivePage } from '@bifrost-orbit-cms/sdk/react'
import { registry } from '@/cms/registry'

export function LivePage({ page }: { page: CmsPageData }) {
  return <CmsLivePage page={page} registry={registry} />
}
```

```tsx
// components/page-body.tsx
import type { CmsPage as CmsPageData } from '@bifrost-orbit-cms/sdk'
import { CmsPage } from '@bifrost-orbit-cms/sdk/react'
import { registry } from '@/cms/registry'
import { LivePage } from './live-page'

export function PageBody({ page }: { page: CmsPageData }) {
  if (page.draft) return <LivePage page={page} />
  return <CmsPage page={page} registry={registry} />
}
```

- Components then also run in the browser while previewing: keep them free of server-only code such as secrets or file reads.
- `useLivePage(page)` gives the live page when you render more than the blocks, such as a content type's template. `onSaved` is called when the editor saves or publishes.
- `CmsLivePage` and `useLivePage` (SDK 0.10 or later) also run the editor's accessibility checks on the rendered page; nothing else to set up.

## Refresh when editors publish

```ts
// app/api/revalidate/route.ts
import { createRevalidateHandler } from '@bifrost-orbit-cms/sdk/next'

export async function POST(request: Request): Promise<Response> {
  const secret = process.env.CMS_WEBHOOK_SECRET
  if (!secret) return Response.json({ error: { code: 'not_configured' } }, { status: 503 })
  return createRevalidateHandler({ secret })(request)
}
```

It checks the `x-orbit-signature` header, refuses messages with a bad signature or older than five minutes with `401`, and refreshes the cache tags of what changed. The person then adds a webhook under **Configuration › Webhooks** for the Production environment, ticks **Page deleted** and **Release published** as well as the defaults, and puts its `whsec_` secret in `CMS_WEBHOOK_SECRET`. **Send a test** in the webhook's menu answers `{"revalidated":[]}`.

Tags the client sets: `cms` on everything, `cms:page:<region>:<path>`, `cms:nav:<key>`, `cms:global:<key>`, `cms:list` and more; `cacheTags` from `@bifrost-orbit-cms/sdk` builds them for your own `fetch` calls. On other frameworks, check the message with `verifyWebhook({ body, timestamp, signature, secret })` and use its `tags`.

## Common mistakes

- A secret `cms_sk_` key in browser code. `previewKey` is refused there; keep the client in a `server-only` file.
- Reading drafts without `preview: true`, or caching drafts. Previews are never cached by the SDK; do not add your own cache for them.
- Forgetting `data-cms-id`, so blocks cannot be clicked in the preview.
- Linking with `link.href` without checking for `null`.
- Passing a region's address prefix or short code instead of its key.
- Hard-coding the component list from memory. Read the schema, generate types and use the space's keys.
- Editing `cms/types.ts` by hand. Generate it.

## Docs

- Next.js quickstart: https://docs.bifrostorbit.com/cms/developers/quickstart-nextjs/
- Astro quickstart: https://docs.bifrostorbit.com/cms/developers/quickstart-astro/
- SDK reference: https://docs.bifrostorbit.com/cms/developers/sdk/
- Render components: https://docs.bifrostorbit.com/cms/developers/render-blocks/
- Live preview: https://docs.bifrostorbit.com/cms/developers/preview/
- Caching and refreshing: https://docs.bifrostorbit.com/cms/developers/caching-and-refresh/
- Webhooks: https://docs.bifrostorbit.com/cms/developers/webhooks/
- Delivery API: https://docs.bifrostorbit.com/cms/developers/delivery-api/
