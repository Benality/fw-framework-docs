# Moving to Mintlify

This repo (`fw-framework-docs`) was converted from MkDocs Material to Mintlify. This file is the one-time cutover checklist. Delete it once the cutover is done and the team has read it, or keep it as a record — your call.

## What already happened

- All 16 pages moved from `docs/content/*.md` to `docs/*.mdx`, each with `title` + `description` frontmatter and no duplicate `#` H1 (the frontmatter title renders that).
- Internal links rewritten from `fw-core.md` / `fw-core.md#account` style to Mintlify's extensionless `/fw-core` / `/fw-core#account` style.
- `docs/docs.json` replaces `docs/mkdocs.yml` — same two tabs (Reference, Tutorials), same page order.
- Brand assets moved: `docs/content/assets/images/territory-crest.webp` → `docs/images/territory-crest.webp`, `docs/content/assets/fonts/house-of-cards-bold.ttf` → `docs/fonts/house-of-cards-bold.ttf`.
- Removed: `mkdocs.yml`, `requirements.txt`, `docs/content/`, `.github/workflows/publish.yml`, `site/` (gitignored build output).
- `npx mint broken-links` passes clean against the new structure.
- Updated `.cursor/rules/publish-docs.mdc` and `.cursor/rules/developer-docs.mdc` so future doc edits (by a human or an agent) target the new `.mdx` files and `docs.json`, not the retired mkdocs paths.
- Updated every `resources/[framework]/*/README.md` that pointed at `docs/content/fw-*.md` to point at `docs/fw-*.mdx` instead.

## What's left — do this to go live

1. **Create the Mintlify project.** Sign in at [mintlify.com](https://mintlify.com) with the account that should own this docs site, and connect it to the `Benality/fw-framework-docs` GitHub repo. Mintlify's GitHub App watches `master` — once connected, every push deploys automatically. No workflow file needed; that's why `.github/workflows/publish.yml` was deleted.

2. **First deploy and visual QA.** After connecting, Mintlify builds `master` as-is. Open the generated preview URL and check:
   - Both tabs (Reference, Tutorials) and all 16 pages render.
   - Logo and favicon show up (`territory-crest.webp` — if Mintlify's image pipeline rejects `.webp` for the favicon specifically, re-export it as a `.png` and point `favicon` in `docs.json` at that instead; the logo field is more lenient about format).
   - Dark theme applies by default (`appearance.strict: true` in `docs.json` hides the light/dark toggle entirely, so there's no light-mode variant to check).
   - You can also run this locally first with `npx mint dev` from `docs/` — it needs network access to Mintlify's preview service, so it won't work in a fully offline/sandboxed shell.

3. **Swap in the real display font.** `docs.json` currently points `fonts.heading.family` at **Rye**, a Google Font stand-in — it's a reasonable match for a western wanted-poster look but it is not the site's actual "House of Cards" face. Mintlify's `fonts.source` only accepts `woff`/`woff2`, and the original asset is a `.ttf` (now at `docs/fonts/house-of-cards-bold.ttf`). Convert it once (e.g. `npx @fontsource/cli` is not it — use something like [transfonter.org](https://transfonter.org) or local `fonttools`/`woff2_compress`), commit the `.woff2` next to the `.ttf`, and update:
   ```json
   "fonts": {
     "heading": { "family": "House of Cards", "source": "/fonts/house-of-cards-bold.woff2", "format": "woff2" }
   }
   ```

4. **Known branding gap: no custom CSS.** The old `stylesheets/extra.css` did a few things Mintlify's `docs.json` has no equivalent for — a rounded logo mask, a hairline border under the header, pill-shaped nav tabs. `theme`, `colors`, `fonts`, and `background` in `docs.json` get you close (dark background, gold accent, custom heading font) but not pixel-identical. There's no supported custom-CSS injection in Mintlify's config today. If that gap matters, the options are: live with it, or revisit per-component theming once Mintlify ships it. Try the other 8 theme values (`mint`, `maple`, `palm`, `linden`, `almond`, `aspen`, `sequoia`, `luma` vs. the currently-set `willow`) with `npx mint dev` — pick whichever reads best with this palette before going live.

5. **Custom domain (optional).** The retired site had no custom domain — it was plain `benality.github.io/fw-framework-docs/`. If you want `docs.territoryrp.com` (or similar) on Mintlify instead of their default `*.mintlify.app` subdomain, set it up in the Mintlify dashboard under Settings → Domain, then add the CNAME record it gives you at your DNS provider. This is independent of the rest of the cutover and can happen before or after step 6.

6. **Cut over and retire the old site.**
   - Once the Mintlify deploy looks right, that's the live docs site — share its URL with the team and update any links pointing at `benality.github.io/fw-framework-docs`.
   - GitHub Pages for this repo is served off the `gh-pages` branch, built by the now-deleted `publish.yml`. Nothing pushes to `gh-pages` anymore, so it will simply go stale. Once you're confident in the Mintlify deploy (give it a few days), disable Pages for the repo (Settings → Pages → set Source to "None") so the stale build doesn't linger as a second, outdated copy of the docs.
   - URL shapes differ slightly: MkDocs Material built `/fw-core/index.html` (served at `/fw-core/`, trailing slash); Mintlify serves `/fw-core` (no trailing slash, but it also answers the trailing-slash form). Existing inbound links shouldn't break, but if you find one that 404s, add it to `redirects` in `docs.json`.

7. **Rollback, if needed.** The old mkdocs files are only gone from the working tree — they're still in this repo's git history on `master` (and still exist if you check out an earlier commit). If Mintlify turns out to be a dead end, `git revert` the cleanup commit(s), restore `gh-pages` from history, and re-run `mkdocs gh-deploy`. Nothing about this migration was destructive at the git level.

## Day-to-day workflow after this

Unchanged in spirit, changed in mechanics: edit the relevant `docs/*.mdx` file, add any new page's slug to the matching group's `pages` array in `docs/docs.json`, commit, push to `master`. Mintlify deploys on its own — there's no `mkdocs gh-deploy` step to remember or forget. `.cursor/rules/publish-docs.mdc` has the short version.
