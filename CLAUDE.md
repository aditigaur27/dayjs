# Day.js

Immutable date library with a Moment.js-compatible API, kept under a 2.99 KB gzipped size budget.
The core is deliberately small; nearly every feature beyond parse/format/manipulate/compare ships as an
opt-in plugin, and every language ships as an opt-in locale.

## Layout

- `src/index.js` — the whole core: the `Dayjs` class, the `dayjs()` factory, `dayjs.extend`, `dayjs.locale`.
- `src/utils.js`, `src/constant.js` — shared helpers (`Utils` is exposed to plugins) and unit/regex constants.
- `src/plugin/<name>/index.js` — one directory per plugin, always `export default (option, Dayjs, dayjs) => {}`.
  Plugins add behavior by patching `Dayjs.prototype` (often wrapping the old method) or static `dayjs.*`.
- `src/locale/<code>.js` — one file per locale; must contain a `// Language Name [code]` comment, by
  convention line 1 (`build/index.js` and `test/locale/keys.test.js` both regex it out).
- `types/` — hand-written TypeScript declarations; `types/plugin/<name>.d.ts` per plugin.
- `test/` mirrors `src/`: `test/<area>.test.js`, `test/plugin/<name>.test.js`, `test/locale/<code>.test.js`.

## Commands

```bash
npm ci                                                 # install (lockfile is committed)
npx jest test/plugin/isToday.test.js --coverage=false  # one test file, fast
TZ=UTC npm test                                        # full suite: tz tests in 7 pinned zones + host zone, then everything with 100% line coverage enforced
TZ=Pacific/Auckland npx jest test/timezone.test.js --coverage=false  # re-run one failing tz step in its zone
npm run lint                                           # eslint over src/ test/ build/ (also a pre-commit hook)
npx eslint src/plugin/isToday/index.js                 # lint one file
NODE_OPTIONS=--openssl-legacy-provider npm run build   # rollup to repo root, then size-limit check (2.99 KB)
```

- Always prefix full runs with `TZ=UTC`. The last `npm test` step runs in the host zone, and on a US-timezone
  machine 6 tests in `dayOfYear`, `localizedFormat` and `timezone` fail (host DST shifts; `dayjs(0)` formats as 1969).
  CI runs in UTC.
  Those failures are environmental; don't "fix" source or tests for them.
- On Node 17+, `npm run size` (inside `build`) dies with `ERR_OSSL_EVP_UNSUPPORTED` from size-limit's bundled
  webpack 4. The rollup output is already written by then; the env var above makes the size check run.

## Conventions

- Style is airbnb-base with **no semicolons** and **no trailing commas**.
- Tests compare against Moment as the oracle: `expect(dayjs(x).format(f)).toBe(moment(x).format(f))`.
  Pin "now" with `MockDate.set(new Date())` in `beforeEach` and `MockDate.reset()` in `afterEach`.
- Test files import from source, never the build: `import dayjs from '../../src'`,
  `import plugin from '../../src/plugin/<name>'`, then `dayjs.extend(plugin)` at top level.
- Private instance state is `$`-prefixed (`$d` native Date, `$L` locale, `$u` UTC flag, `$x` plugin extras).
- Coverage must stay at 100% of lines in `src/` — every new branch needs a test or `npm test` fails.
- PRs target `dev` (CI runs there); pushes to `master` trigger an automatic semantic-release publish.

## Guardrails

- A new plugin needs three files: `src/plugin/<name>/index.js`, `types/plugin/<name>.d.ts` (enforced by
  `test/plugin/declarations.test.js`), and `test/plugin/<name>.test.js`.
- A new locale must pass `test/locale/keys.test.js`: lowercase `name` equal to the filename, an `ordinal`
  function; if present, `formats` has exactly `LT LTS L LL LLL LLLL` (+ optional lowercase) and `relativeTime`
  has all 13 keys.
- Do not edit or commit build output at the repo root: `dayjs.min.js`, `/plugin`, `/locale`, `index.d.ts`,
  `locale.json` (from `npm run build`) and `/esm` (from `npm run babel`) are gitignored. Edit `src/` and `types/`.
- Locale files `import dayjs from 'dayjs'`; in tests that resolves to `src` via `test/__mocks__/dayjs.js`.
  Don't "fix" that import to a relative path.
- Don't add core features to `src/index.js` that could be a plugin; `npm run build` fails past 2.99 KB.
- `docs/` holds translated READMEs and stub pages that redirect to day.js.org (a separate repo). Don't
  document API changes there.
- `npm run sauce` / `test:sauce` need Sauce Labs credentials; don't run them.
