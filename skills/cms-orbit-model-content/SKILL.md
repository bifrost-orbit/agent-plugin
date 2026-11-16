---
name: cms-orbit-model-content
description: Use when creating or changing the schema of a CMS Orbit space, its content types, components (blocks) and fields, or when designing a content model from a brief, a design or an existing website. Covers schema files with the CLI (schema init, validate, diff, push, rollback), field types, slots, links, references, showWhen and reusing blocks. For example "add a season field to the trip hero", "model a journal with authors", "set up blocks for this landing page design".
---

# Model content in CMS Orbit

The schema is the space's **content types** and **components**. It belongs to the space, so every environment (Staging, Production) shares it. Keep it in the repository as TypeScript files and change it with the CMS Orbit CLI, so changes are reviewed in pull requests next to the code that renders them.

The MCP server cannot change the schema; agents only read it there with `get_schema`. Use the CLI. The person needs a role with `schema:manage` (Developer, Admin or Owner) to push. See `cms-orbit-use-cli` for signing in and choosing a space.

## Steps, in order

1. **Get the current schema into files**, unless `cms/schema/` already holds it:
   ```bash
   npx cms-orbit schema init
   ```
   This writes `cms/schema/content-types/<key>.ts`, `cms/schema/components/<key>.ts` and `cms/schema/index.ts`. It refuses to overwrite a folder that holds a schema unless you pass `--force`; if the folder exists, run `schema diff` first to see whether it is current.
2. **Look for what already exists.** Read the files, or the space's overview with the MCP tool `get_schema`. Reuse a component that does the job, or add a field or a choice to it, before you create a new one. A `textSection` with a heading level serves more pages than `introSection`, `outroSection` and `sidebarSection`.
3. **Say what you will add or change**, field by field, before editing.
4. **Edit the files.** One definition per file, exported as default.
5. **Check them** without changing anything:
   ```bash
   npx cms-orbit schema validate
   ```
   It exits with `1` on an error. `--remote` also checks the field types against the server.
6. **See the change** against the space:
   ```bash
   npx cms-orbit schema diff --uses
   ```
   `--uses` counts the entries holding values in anything that would be removed.
7. **Push it** after the person has seen the diff:
   ```bash
   npx cms-orbit schema push
   ```
   It shows the diff, asks, saves a snapshot and prints the rollback command. `--dry-run` shows the diff and stops. Only pass `--yes` when the person asked you to apply it without a question, or in CI.
8. **Update the types** for the website, straight from the files, so they follow the schema in the same commit:
   ```bash
   npx cms-orbit types generate --from cms/schema --out src/cms/types.ts
   ```
9. **Undo** if needed: `npx cms-orbit schema rollback` lists the snapshots, and `npx cms-orbit schema rollback latest` puts back what the last push changed.

## A component file

```ts
// cms/schema/components/trip-hero.ts
import type { CmsComponentDefinition } from "@bifrost-orbit-cms/sdk";

export default {
  key: "trip-hero",
  name: "Trip hero",
  group: "Trips",
  fields: [
    { key: "heading", name: "Heading", type: "text", required: true },
    { key: "season", name: "Season", type: "select", options: [{ label: "Summer", value: "summer" }, { label: "Winter", value: "winter" }] },
    { key: "photo", name: "Photo", type: "asset" },
  ],
  mainHeading: "heading",
} satisfies CmsComponentDefinition;
```

A content type file uses `CmsContentTypeDefinition`:

```ts
// cms/schema/content-types/journal-post.ts
import type { CmsContentTypeDefinition } from "@bifrost-orbit-cms/sdk";

export default {
  key: "journalPost",
  name: "Journal post",
  urlPrefix: "/journal",
  fields: [
    { key: "excerpt", name: "Excerpt", type: "textarea", validation: { maxLength: 300 } },
    { key: "publishedOn", name: "Published on", type: "date" },
    { key: "author", name: "Author", type: "reference", allowedContentTypes: ["author"] },
    { key: "cover", name: "Cover image", type: "asset" },
    { key: "topics", name: "Topics", type: "taxonomyTerms", taxonomyKey: "topics" },
  ],
  allowedComponents: ["text", "textSection", "imageBlock", "gearList", "sharedSection"],
} satisfies CmsContentTypeDefinition;
```

A file may import types but no code: each file is read on its own. Settings left at their defaults are left out.

## Field types

| Type | Holds | Settings it needs |
| --- | --- | --- |
| `text`, `textarea` | Text, one line or several | `validation.maxLength`, `pattern` |
| `richtext` | HTML with headings, lists, links and tables. No images | |
| `number` | A number | `validation.min`, `max` |
| `boolean` | On or off | |
| `select`, `multiselect` | One or several of a fixed list | `options: [{ label, value }]` |
| `color` | A colour, as a hex string | `options` to offer a set of colours |
| `date`, `time`, `datetime` | ISO 8601 text | |
| `asset`, `assets` | One or several media files | `allowedMimeTypes`, such as `["image/*"]` |
| `link` | Where a button or card goes: a page, an address, an email or a file | |
| `targets` | Pages and sections, as addresses | |
| `reference`, `references` | Other entries with a life of their own | `allowedContentTypes` |
| `taxonomy`, `taxonomyTerms` | Terms from a taxonomy editors maintain | `taxonomyKey` |
| `claims` | Approved claims | |
| `component`, `components` | Slots: blocks nested in this block | `allowedComponents` |
| `trip`, `trips`, `destination`, `departure` | Picks from Travel Orbit, when the space is connected | |
| `product`, `products`, `collection` | Picks from Commerce Orbit, when connected | |
| `bookingService`, `bookingServices`, `bookingStaff` | Picks from Booking Orbit, when connected | |
| `reviewLocation`, `job`, `jobFilter` | Picks from Presence Orbit and Careers Orbit | |

Settings every field can have: `key`, `name`, `type`, `description`, `required`, `default`, `validation`, `regional` (keeps its own value in each region when a region's copy is updated, for prices or phone numbers), `protects` (`"price"` or `"promotion"`; agents can never change these), `group`, `hidden` and `showWhen`.

Field reference with every setting and how each arrives in the delivery API: https://docs.bifrostorbit.com/cms/developers/field-types/

## Choosing the building block

- **Content type** if it has its own address, appears in lists or is published on its own: a journal post, a trip report, an author. Keep its own fields to what lists, cards, feeds and search need. The page body is blocks.
- **Component** if it only exists as part of a page: a hero, a text section, a gear list.
- **Slot** (`component` or `components` field with `allowedComponents`) for blocks that belong to the parent and this page only, such as the items in a gear list. Give child-only components `isRoot: false` so they are offered only inside their parent.
- **Reference** for an entry used in many places whose fields the website needs, such as a post's author.
- **Link** to send a visitor to a page. It follows the page when it moves.
- **Taxonomy** for choices editors may extend, such as topics or activities. **Select** for choices the code depends on, such as a heading level or a layout.
- **Shared section** for blocks reused on many pages and edited once: allow `sharedSection` in `allowedComponents`.
- **Global entry** (`isGlobal: true`) for content shown once per website, such as the header or footer.

Regions and languages are not part of the model: every type and component works in every region and language.

## showWhen

A field can show only while another field of the same type or component holds a value. Only `select`, `multiselect`, `boolean`, `text` and `textarea` fields can decide:

```ts
{ key: "kind", name: "Goes to", type: "select", options: [{ label: "Sign up", value: "signUp" }, { label: "A page", value: "page" }] },
{ key: "plan", name: "Plan", type: "select", options: [/* … */], showWhen: { field: "kind", equals: "signUp" } },
```

`equals` is `true` or `false` for a boolean, one value, or a list meaning any of them, such as `["signUp", "signIn"]`. While the condition does not hold, the field is hidden, not required, not checked and not delivered, but keeps its value. A condition on a missing field, or one that makes a loop, is refused.

## Main heading

Each page needs one H1. On a component the website renders as the page's H1, set `mainHeading` to that field, such as `"heading"`. `mainHeadingWhen` names a boolean field when it is the H1 only while that is on. On a content type whose template prints the H1 from a field, set `mainHeading` there. Without it, only components with "hero" in their key count as providing one.

## Rules and common mistakes

- **Keys are for ever.** Renaming a key in a file removes the old field or definition and adds a new one, and the entries' values stay under the old key. Pick keys carefully.
- **Use letters, digits and underscores in field keys**, such as `heroImage`. Keys with hyphens cannot be filtered on in the delivery API and need quotes in code. Component and content type keys may use hyphens, such as `trip-hero`.
- **Name for meaning, not looks:** `gearList`, not `threeColumnGrid`.
- **Images go in `asset` fields, never in rich text.** Rich text removes images.
- **Required means required to publish.** Drafts may leave required fields empty.
- **A push refuses to lose content.** Removing a field that entries hold values in, or changing its type, is refused with the number of entries affected unless you pass `--allow-destructive`. Do not add that flag on your own: tell the person what would be lost and let them decide. A migration (see `cms-orbit-use-cli`) can move the values to the new field first.
- **A push never removes a content type or component that entries still use.** Retire one with `deprecated: true` instead; deprecated components are left out of `get_schema` and of generated types.
- **Copying a schema between spaces:** `npx cms-orbit schema push --space northwilds-test --from-space northwilds`.
- **Components alone, as JSON:** `npx cms-orbit components pull --only trip-hero` and `components push --only trip-hero --dry-run`, for teams that don't keep the whole schema in TypeScript. `components push` leaves components without a file alone unless you pass `--delete-missing`.

## Docs

- Model your content: https://docs.bifrostorbit.com/cms/developers/model-your-content/
- Field types: https://docs.bifrostorbit.com/cms/developers/field-types/
- Schema commands: https://docs.bifrostorbit.com/cms/developers/cli/#schema
- Design components and sections: https://docs.bifrostorbit.com/cms/developers/design-components/
