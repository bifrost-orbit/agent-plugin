---
name: cms-orbit-use-cli
description: Use when running the CMS Orbit CLI, npx cms-orbit, for anything outside the editor. Covers signing in, choosing the space, environment, region and language, schema and components, types, languages, pulling and pushing content, media and taxonomies as files, migrations with generate, run, rollback and list, starting a project, logs, reports and CI. For example "pull all the trips as JSON", "rename a value in every hero block", "copy the schema to the test space", "start a Next.js site for our space".
---

# Drive CMS Orbit with the CLI

`cms-orbit` comes with the JavaScript SDK, `@bifrost-orbit-cms/sdk` 0.16.0 or later. Run it with `npx cms-orbit`, or add the SDK to the project and call it from `package.json` scripts to pin a version. It needs Node.js where it runs, not in the website, so PHP, Swift and Kotlin projects use it too.

`npx cms-orbit --help` lists every command; `--help` after any group or command shows its options. Read it when you are not sure of a flag rather than guessing.

## Sign in first

```bash
npx cms-orbit login     # opens the browser; the person signs in and selects Allow
npx cms-orbit user      # who is signed in, their organisations and the server
```

- `login` is for the person to run: it opens their browser and waits. Over SSH, `--no-browser` prints the link instead. Do not ask the person for their password or a token.
- The CLI then **acts as that person**, with their role in every organisation they belong to. Its changes are recorded as them, marked "(command line)".
- In CI, use a management token: `ORBIT_TOKEN=cms_mt_…` in the environment, or `cms-orbit login --token -` to read one from standard input. A token always wins over a stored sign-in.
- Staging of CMS Orbit itself: `ORBIT_API=https://api-stage.bifrostorbit.com/cms/manage/v1`.

## Choose where it works

Commands that work in a space take `--organisation`, `--space`, `--environment`, `--region` and `--locale`, each by id or by the name you'd type (slug, key or code). Defaults: the only organisation or `ORBIT_ORGANISATION`, the only space or `CMS_SPACE_ID` / `ORBIT_SPACE`, **Staging**, the first region and its default language. The command prints what it chose, such as `Using Northwilds › Northwilds › staging › europe › en-GB`; check that line.

- **Nothing changes in Production unless you pass `--environment production`.** Only do that when the person asked.
- The **schema belongs to the space**, so `schema` and `components` commands take no `--environment`: a schema push changes every environment at once.

## Safe habits

1. Read first: `content find`, `schema diff`, `languages list`.
2. Run the change with `--dry-run` and show the person the result.
3. Run it for real only when they agree. Never add `--yes`, `--force`, `--allow-destructive`, `--delete-missing`, `--remove-missing` or `--publish` on your own; each one removes a safeguard. Say what it would do and let the person decide.
4. Add `--json` when you need to read the answer: it prints only JSON on standard output.
5. Exit codes: `0` done, `1` failed (refused, not signed in, validation errors), `2` wrong command or options. Errors start with `cms-orbit:` and say what to do next.

## Commands

| Group | Commands |
| --- | --- |
| Account | `login`, `login --token <token>`, `logout` (`--all`), `user`, `signup` |
| `schema` | `init`, `validate`, `diff`, `push`, `rollback`: see `cms-orbit-model-content` |
| `components` | `pull`, `push` (`--only trip-hero,gear-list`, `--delete-missing`) |
| `types` | `generate` (`--lang ts\|php\|swift\|kotlin`, `--out`, `--from cms/schema`, `--include references`) |
| `languages` | `list`, `pull` (writes `cms/languages.json`) |
| `content` | `find`, `pull`, `push`, `validate` |
| `media` | `pull`, `push` |
| `taxonomies` | `pull`, `push`, `delete` |
| `migrations` | `generate`, `run`, `rollback`, `list` |
| `project` | `create` |
| `logs` | `list` (`--space`, `--since 7d`, `--limit`), `prune` (`--keep`, 30 days by default) |
| `reports` | `list`, `prune` |
| Older commands | `generate` (same as `types generate`), `import <file>`, `promote` |

Docs for every command: https://docs.bifrostorbit.com/cms/developers/cli/

## Content as files

```bash
npx cms-orbit content find --type trip --meaning "paddling between islands" --limit 5
npx cms-orbit content find --status draft --since 2d --json
npx cms-orbit content pull --folder trips --all-locales
npx cms-orbit content validate cms/content
npx cms-orbit content push --dry-run
npx cms-orbit content push
```

- `find` filters with `--type`, `--folder`, `--path /trips/`, `--status`, `--since`, `--text` (add `--deep` to search the fields) and `--meaning` (Gravity, when on).
- `pull` writes one file per entry to `cms/content/<region>/<language>/<content folder>/<slug>.json` with its `id`, `docVersion`, `fields`, `components` and `seo`, and a `.cms-orbit.json` saying which space and environment it came from. `--published` adds what is live, for reference only.
- **Edit the files, keep `id` and `docVersion`.** `push` saves drafts. An entry someone changed since the pull is a **conflict**: nothing is saved for it. Pull it again and redo the change; never work around a conflict.
- A file without an `id` is matched by its address, then its external ID, or created as a new draft. Its content folder must exist.
- `push` refuses a folder pulled from another space or environment unless you pass `--force`.
- `--publish` publishes what it saved, where the person may publish. Leave it off unless asked.

## Migrations: change many entries in one go

```bash
npx cms-orbit migrations generate hero
```

This writes `cms/migrations/<date>-hero.ts` with the component's fields typed. Fill in the function: for a component it is called with every block of that type, at any depth, and returns the changed block or `null` to remove it. For a content type (`--kind content-type`) it gets each entry and returns it with changed `fields`, `components` or `seo`.

```ts
// cms/migrations/2026-11-16-hero.ts (generated, then filled in)
export const target = { component: "hero" } as const;

export default function migrate(block: Block<HeroFields>): Block<HeroFields> | null {
  if (block.fields.size === "large") block.fields.size = "full";
  return block;
}
```

```bash
npx cms-orbit migrations run cms/migrations/2026-11-16-hero.ts --dry-run
npx cms-orbit migrations run cms/migrations/2026-11-16-hero.ts
npx cms-orbit migrations list
npx cms-orbit migrations rollback last --dry-run
npx cms-orbit migrations rollback last
```

- `run` works in one language unless you pass `--all-locales`; `--type` limits a component migration to entries of some content types.
- A real run saves a snapshot of every draft first, saves each with the version it read (a concurrent change is a conflict, not overwritten), and prints the run's id. Nothing is published unless `--publish`.
- `rollback` takes the run id, its end, or `last`. Entries changed since the run are left alone unless `--force`. Rolling back never publishes.
- Migrations need Node.js 22.18 or later to run TypeScript; on older versions write them as `.mjs`.
- To rename a field safely: add the new field in the schema, push, migrate the values across, check, then remove the old field.

## Media and taxonomies

```bash
npx cms-orbit media pull --folder Trips
npx cms-orbit media push ./photos/lofoten-ridge.jpg --folder "Trips"
npx cms-orbit taxonomies pull
npx cms-orbit taxonomies push --dry-run
```

- `media pull` writes the library to `cms/media` with `media.json` holding alt text, titles, captions, tags and focus points. `media push` uploads what the library doesn't have and saves changed metadata; files are matched so nothing is uploaded twice. Give every image alt text.
- `taxonomies pull` writes `cms/taxonomies/<key>.json` (`--csv` for spreadsheets). `push` creates and updates terms by value; `--remove-missing` deletes terms the file leaves out.
- `taxonomies delete <key>` asks the person to type the key, refuses while a field uses the taxonomy unless `--force`, and keeps a copy that `push` puts back. Taxonomies have no Bin; leave deleting to the person.

## Start a project

```bash
npx cms-orbit project create northwilds-site --framework nextjs --create-key
cd northwilds-site && npm install && npm run dev
```

Frameworks: `nextjs`, `react`, `astro`, `nuxt`, `vue`, `sveltekit`, `angular`, `laravel`, `symfony`. `--template website` gives the ready-made Next.js website for the space templates. It writes the env file with the Delivery API address, the region and a public key for Staging (`--environment`, `--region`, `--public-key cms_pk_…` or `--create-key`). `--install` installs dependencies. Then load `cms-orbit-build-frontend`.

## CI

```bash
ORBIT_TOKEN=cms_mt_… npx cms-orbit schema diff --exit-code
ORBIT_TOKEN=cms_mt_… npx cms-orbit content validate --strict
```

`schema diff --exit-code` fails when files and space differ, and `content validate` fails on errors (warnings too with `--strict`). Requests that meet `429` or `503` are retried, and every change carries an `Idempotency-Key`, so a retried change is made once.

## After a change

`npx cms-orbit logs list` shows what the command line changed, and `npx cms-orbit reports list` where each bulk command wrote its report. Tell the person what changed and where the report is.
