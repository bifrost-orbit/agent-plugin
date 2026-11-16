# CMS Orbit with Astro

Astro uses the SDK's core client only; React is not needed. The website renders on the server with the Node.js adapter, so new pages appear without a rebuild. Astro 7 needs Node.js 22.23 or later. The finished code is the starter `bifrost-orbit/cms-starter-astro`, or `npx cms-orbit project create northwilds-site --framework astro`.

Full guide: https://docs.bifrostorbit.com/cms/developers/quickstart-astro/

## Settings

`.env`, with the same keys as for Next.js (`CMS_API_URL`, `CMS_PUBLIC_KEY`, `CMS_PREVIEW_KEY`, `CMS_DEFAULT_REGION`, `CMS_WEBHOOK_SECRET`). Declare them in `astro.config.mjs`, so secrets stay out of browser code:

```js
// astro.config.mjs
import { defineConfig, envField, memoryCache } from 'astro/config'
import node from '@astrojs/node'

export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),
  cache: { provider: memoryCache() },
  env: {
    schema: {
      CMS_API_URL: envField.string({ context: 'server', access: 'public' }),
      CMS_PUBLIC_KEY: envField.string({ context: 'server', access: 'secret' }),
      CMS_PREVIEW_KEY: envField.string({ context: 'server', access: 'secret', optional: true }),
      CMS_WEBHOOK_SECRET: envField.string({ context: 'server', access: 'secret', optional: true }),
      CMS_DEFAULT_REGION: envField.string({ context: 'server', access: 'public', default: 'global' }),
    },
  },
})
```

## Client and preview check

```ts
// src/lib/cms.ts
import { createCmsClient } from '@bifrost-orbit-cms/sdk'
import { CMS_API_URL, CMS_DEFAULT_REGION, CMS_PREVIEW_KEY, CMS_PUBLIC_KEY } from 'astro:env/server'

export const cms = createCmsClient({
  baseUrl: CMS_API_URL,
  apiKey: CMS_PUBLIC_KEY,
  previewKey: CMS_PREVIEW_KEY,
  defaultRegion: CMS_DEFAULT_REGION,
})

export async function isPreview(url: URL): Promise<boolean> {
  if (!CMS_PREVIEW_KEY) return false
  const check = await cms.checkPreview(url.searchParams.get('preview'))
  return check.valid
}
```

## The page route

```astro
---
// src/pages/[...slug].astro
import { cacheTags } from '@bifrost-orbit-cms/sdk'
import Layout from '../layouts/Layout.astro'
import Block from '../components/Block.astro'
import { cms, isPreview } from '../lib/cms'

const path = '/' + (Astro.params.slug ?? '')
const preview = await isPreview(Astro.url)
const result = await cms.resolve({ path, preview })

if (result.kind === 'redirect') return Astro.redirect(result.to, result.status as 301 | 302 | 307 | 308)
if (result.kind === 'gone') return new Response(null, { status: 410 })
if (result.kind === 'notFound') return new Response(null, { status: 404 })

const page = result.page
if (!preview) {
  Astro.cache.set({ maxAge: 60, swr: 60, tags: [cacheTags.all, cacheTags.page(page.region, path)] })
}
---

<Layout title={page.seo.title} description={page.seo.description} lang={page.locale} preview={preview}>
  <main>
    {page.components.map((block) => <Block block={block} />)}
  </main>
</Layout>
```

In the layout, load the bridge only while previewing: `{preview && <script is:inline src={`${cms.origin}/bridge.js`} defer></script>}`. The token stays in the address, so nothing is stored in a cookie, and drafts skip `Astro.cache.set`.

## Blocks

One component looks up the rest:

```astro
---
// src/components/Block.astro
import type { CmsComponent } from '@bifrost-orbit-cms/sdk'
import { registry } from './registry'

interface Props { block: CmsComponent }
const { block } = Astro.props
const Component = registry[block.type]
---

{Component ? <Component block={block} /> : import.meta.env.DEV && <p data-cms-id={block.id}>No component for <code>{block.type}</code> yet.</p>}
```

```ts
// src/components/registry.ts
import Hero from './Hero.astro'

export const registry: Record<string, (props: { block: any }) => unknown> = { hero: Hero }
```

In Astro the values are in `block.fields`, typed with `CmsNode<HeroBlock>` from the generated file (`"cms:types": "cms-orbit generate --out src/cms/types.ts"`). Rich text is HTML, set with `set:html`. Nested blocks in a slot are drawn with `Block` again:

```astro
---
// src/components/Hero.astro
import type { CmsNode } from '@bifrost-orbit-cms/sdk'
import type { HeroBlock } from '../cms/types'
import Block from './Block.astro'

interface Props { block: CmsNode<HeroBlock> }
const { block } = Astro.props
const { heading, description, primaryAction } = block.fields
---

<section data-cms-id={block.id}>
  <h1>{heading}</h1>
  {description && <div set:html={description} />}
  {primaryAction && <Block block={primaryAction} />}
</section>
```

Images: `cmsImageSrcSet(image.url, image.width)` from `@bifrost-orbit-cms/sdk` for `srcset`, with `image.width`, `image.height` and `image.alt` on the `<img>`.

## Webhook

```ts
// src/pages/api/cms-webhook.ts
import type { APIRoute } from 'astro'
import { cacheTags, verifyWebhook } from '@bifrost-orbit-cms/sdk'
import { CMS_WEBHOOK_SECRET } from 'astro:env/server'

export const POST: APIRoute = async ({ request, cache }) => {
  if (!CMS_WEBHOOK_SECRET) return new Response(null, { status: 401 })
  const result = await verifyWebhook({
    body: await request.text(),
    timestamp: request.headers.get('x-orbit-timestamp'),
    signature: request.headers.get('x-orbit-signature'),
    secret: CMS_WEBHOOK_SECRET,
  })
  if (!result.valid) return new Response(null, { status: 401 })
  if (result.event === 'webhook.test') return new Response(null, { status: 204 })

  const pageTag = result.tags.find((tag) => tag.startsWith('cms:page:'))
  await cache.invalidate({ tags: pageTag ?? cacheTags.all })
  return new Response(null, { status: 204 })
}
```

Register it under **Configuration › Webhooks** for Production, with **Page deleted** and **Release published** ticked, and put the `whsec_` secret in `CMS_WEBHOOK_SECRET`.

To redraw as editors type rather than after each save, see "Show changes as editors type" in https://docs.bifrostorbit.com/cms/developers/preview/ (`window.cmsOrbit.on('page', …)` once the bridge has loaded).
