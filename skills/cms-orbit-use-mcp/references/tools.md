# CMS Orbit MCP tools

The 29 tools of `https://app.bifrostorbit.com/cms/mcp`. "Scope" means the optional `environment`, `region` and `locale` arguments. Every tool answers with JSON as text; a failure is marked as an error and says why.

Full reference: https://docs.bifrostorbit.com/cms/developers/mcp-tools/

## Find and read (change nothing)

| Tool | Arguments | Answers |
| --- | --- | --- |
| `list_scopes` | none | Each environment with its regions and languages, and `writable` for each environment and region |
| `search_content` | `query` (up to 200 characters, leave out to list all), `mode` (`hybrid` default, `semantic`, `keyword`), `contentType`, `limit` (1 to 100, default 25), scope | `{ total, results: [page summaries with a snippet], gravity: { mode, fallback } }`. Drafts are included. `fallback: true` means Gravity was off and results are keyword matches |
| `find_similar` | `id`, `allLanguages` (default `false`), `limit` (1 to 20, default 8) | `{ results: [{ id, name, url, contentType, locale, score, duplicate }] }`, closest first. Empty with a `note` when Gravity is off |
| `list_pages` | `folderId` (leave out for the top level), scope | `{ folders: [{ id, name, slug }], pages: [page summaries] }` |
| `get_page` | `id` | Name, address, status, content type, folder, the draft's fields, blocks and SEO, draft and published versions, and `docVersion` |
| `get_navigation` | scope | The menus, each with `items` (`label`, `kind`, `visible`, `href`, `page`, `children`). Read only |
| `get_schema` | none for the overview; `contentTypes` (up to 20 keys) and `components` (up to 60 keys) for full definitions | Content types and blocks, the block shape, `unknown` keys, and `travel` and `commerce` sections when the website is connected |
| `list_travel_items` | `kind` (`trips`, `destinations` or `guides`, required), scope | What Trip, Trips, Destination and Departure fields and travel links may pick. Only published, listed trips |
| `list_commerce_items` | `kind` (`products` or `collections`, required), `query`, scope | What Product, Products and Collection fields may pick. Only products on sale |

A page summary is `{ id, name, url, contentType, status, hasUnpublishedChanges, draftedByAi, updatedAt }`. `status` is `draft`, `in_review`, `published`, `scheduled` or `archived`.

## Draft (save drafts)

| Tool | Arguments | Notes |
| --- | --- | --- |
| `create_draft` | `contentType` (required), `name` (1 to 200, required), `slug`, `folderId`, `fields`, `components`, `seo`, scope | Refuses `landingPage`, `popup`, `banner` and `shopProduct`. The address is made from the folder and slug |
| `update_draft` | `id`, `docVersion` (both required), `name`, `fields`, `components` | `fields` and `components` replace the draft's. Answers the new `docVersion` |
| `update_seo_fields` | `id`, `seo` (required), `docVersion` (recommended) | `seo` takes `title` (200), `description` (400), `ogTitle` (200), `ogDescription` (400). Only the fields given change; `null` clears one |
| `draft_special_content` | `id` and `docVersion` to change, or `contentType` and `name` to start; `fields` (only the text fields to change), `components` (whole list, existing blocks kept by `id`), scope | Landing pages, popups, banners and shop products. Wording only |
| `copy_to_region` | `ids` (1 to 50, same region and language), `target: { environment, region, locale }`, `includeMedia` (default `true`) | Linked drafts in another region or language, answered as `pages: [{ id, sourceId, name }]`. Not within the same language |

## Images (change the media library)

| Tool | Arguments | Notes |
| --- | --- | --- |
| `upload_media` | `filename`, `mimeType` (`image/jpeg`, `image/png`, `image/webp`, `image/avif`, `image/gif`), `dataBase64` (at most 3 MB), `alt` by language (required), `title` by language | Into the **AI uploads** folder. JPEG and PNG become WebP (`convertedFrom`, `note`). An image already in the library is returned instead |
| `set_image_text` | `mediaId`, `alt`, `title`, `caption` by language (up to 500 characters each) | Not reviewed, used at once everywhere. `""` or `null` removes a language |

## Checks (change nothing)

| Tool | Arguments | Answers |
| --- | --- | --- |
| `get_seo_issues` | scope | `health`, and `groups`, one per issue `code`, with `level`, `gain` and the pages affected (`wontFix` set where a person decided) |
| `get_accessibility_issues` | scope | Score, guidelines, whether issues warn or block publishing, and open issues by rule with their WCAG criterion |
| `get_page_accessibility` | `id` | `score`, `publish`, and `issues` with `rule`, `severity`, `criterion`, `componentId`, `field`, `foundIn` and `fix` |
| `run_page_checks` | `id` | `seoScore`, `seoIssues`, `accessibilityScore`, `accessibilityIssues` and `publishBlockers` (`required`, `invalid`, `missing_alt`, `missing_subtitles`, `consent`) |

## Review

| Tool | Arguments | Notes |
| --- | --- | --- |
| `add_comment` | `reviewId`, or `pageId` for its latest review; `body` (1 to 4,000, required) | |
| `submit_for_review` | `id`, `message` (1 to 2,000, required), `reviewerIds` (up to 20) | Answers `{ reviewId, status, steps }`. The page then cannot be changed until the review is decided |

## Experiments

| Tool | Arguments | Notes |
| --- | --- | --- |
| `list_experiments` | `pageId`, `status` (`active`, `draft`, `scheduled`, `running`, `paused` or `completed`) | |
| `get_experiment_results` | `id` | `verdict` (`collecting`, `keepGoing`, `winner`, `controlWins`, `noDifference`, `mismatch`), `advice`, variants with rates as fractions |
| `create_experiment` | `pageId`, `name` (1 to 120), `goal` (all required), `hypothesis` (500), `blockId` | Goals: `{ "kind": "click" }`, `{ "kind": "form" }`, `{ "kind": "page", "path": "/book/thank-you/", "match": "exact" }` or `"prefix"`, `{ "kind": "event", "name": "Booking paid" }` |
| `update_experiment_variant` | `id`, `variant` (`b`, `c` or `d`), `components` (required), `fields` | Only before the experiment starts. Answers `changed`, the blocks that differ |

## Tidy up (your own work only)

| Tool | Arguments | Notes |
| --- | --- | --- |
| `discard_draft` | `id` | Moves a page you created through this connection to the Bin. Refused if it was ever published, is in review or a release, is open in the editor, or someone else changed it |
| `discard_upload` | `mediaId` | Moves an image you uploaded to the Bin. Refused while it is used anywhere, and the answer says where |
| `discard_experiment` | `id` | Deletes a draft experiment you created that has not been scheduled or started. Experiments have no Bin |

## Limits

60 requests a minute per connection, 5 MB per request, 3 MB per image, 100 search results, 50 pages per copy, 200-character names, 2,000-character review notes, 4,000-character comments.
