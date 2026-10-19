# KasugaLib-Documentation

The documentation site of the KasugaLib project, published at
<https://kasugalibgroup.github.io/KasugaLib-Documentation/>.

This repository is **deployment only**. It holds no prose. Every page lives in
the main repository and is pulled in at build time.

## How it works

| | |
|---|---|
| Prose lives in | [`KasugaLibGroup/KasugaLibNext`](https://github.com/KasugaLibGroup/KasugaLibNext), under `doc/` |
| Configuration lives here | `mkdocs.yml`, `requirements.txt` |
| Build is driven by | `.github/workflows/deploy-docs.yml` |
| Published to | the `gh-pages` branch, served by GitHub Pages |

A scheduled workflow — roughly every six hours, plus manual runs — fetches
`doc/` from the main repository, builds the site with MkDocs, and pushes the
result to `gh-pages`.

Two things about this setup are deliberate:

- **Nothing is triggered from the main repository.** There is no
  `repository_dispatch` and no push trigger over there. Discovery happens from
  this side, on a schedule.
- **No secrets are needed.** Both repositories are public, so the workflow
  reads the prose unauthenticated and publishes with the token GitHub injects
  into every workflow run.

### Language layout

The documentation is bilingual, and the file layout is what makes it work:

- `page.md` — Chinese. The default language, served from the site root.
- `page.en.md` — English. Served from `/en/`.

The two files sit side by side in the same directory. Links inside the prose
are written **without** a language suffix (`frame-output.md`, not
`frame-output.en.md`); each language resolves to its own version of the target
page.

Note that the Chinese pages carry anchors written in Chinese (`#验证入口`,
`#221-通用关节类型`). Those links only resolve because `mkdocs.yml` sets an
explicit Unicode-preserving `toc.slugify`. Swapping that setting for the
default silently empties every Chinese anchor.

### Change gate

The workflow skips the build when nothing under `doc/` has changed since the
last publish. It records `source_sha` and `built_at` in `site/build-info.json`,
which ships as part of the site — so you can check how fresh a deployment is
without opening the Actions tab.

The gate **fails open**. A missing state file, a first run, an unreachable
comparison endpoint, or a truncated comparison response all mean "build
anyway". A skip is always stated in the workflow log; it is never silent.

### Operational caveats

- GitHub disables scheduled workflows in public repositories after 60 days
  without repository activity. This project deliberately does **not** keep
  itself alive. If the schedule stops firing, run the workflow manually to
  catch up.
- The first deploy with the default token usually fails, because GitHub Pages
  cannot be pointed at a `gh-pages` branch that does not exist yet. The
  recovery order is documented in the header of the workflow file.

## Local development

Requirements: **Python 3.10 or newer.** The pinned Markdown extension pack
dropped support for 3.8 and 3.9.

```bash
git clone https://github.com/KasugaLibGroup/KasugaLib-Documentation
cd KasugaLib-Documentation

python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

The site configuration reads its content from `.docs-src/doc` — a path that
normally does not exist locally, because the workflow is what populates it.
Fetch the prose there yourself, the same way the workflow does:

```bash
git clone --depth 1 \
  https://github.com/KasugaLibGroup/KasugaLibNext \
  .docs-src
```

Then serve the site with live reload at <http://127.0.0.1:8000/>:

```bash
mkdocs serve
```

To produce the same output the workflow publishes:

```bash
mkdocs build          # writes ./site
```

### Things worth knowing before you edit

- **`--strict` is intentionally not used.** The build does not gate on link or
  anchor hygiene, so broken cross-references are possible and will not fail
  anything. Verify them by clicking, not by trusting a green build.
- **Do not add `markdown_extensions` entries casually.** Writing that key at
  all replaces MkDocs' default list, so `toc`, `tables` and the fenced-code
  handling have to be in place explicitly. And `pymdownx.superfences` is a
  *replacement* for `markdown.extensions.fenced_code` — the two cannot be
  loaded together.
- **Every dependency is pinned with `==`.** This repository builds a published
  site, not a library. Upstream releases that are merely newer are not
  automatically something to adopt; change a pin deliberately and say why in
  the comment above it.
- **`nav` is written by hand.** Adding or renaming a page in the main
  repository means editing `nav` here too, along with its `nav_translations`
  entry for the English build. A page missing from `nav` will not appear in
  the sidebar.

`.docs-src/` and `site/` are throwaway build inputs and outputs; both are
git-ignored. To pick up newer prose, delete `.docs-src` and re-run the clone
above, or update it in place with `git -C .docs-src pull`.
