# web_responsive — Odoo 19.0 notes

> Status 2026-06-10: vendored from upstream **OCA/web 19.0** unchanged
> (verified byte-identical at vendoring time), plus **one bringout fix**
> (commit `3a08539`) for the core-version skew described below. Branch
> `19.0` of this oca-web fork (git.hodi.ba/oca/oca-web + github
> bringout/oca-web).

## Symptom

On the v16→v19 migration instance (`bringout-mig-v19-1.hodi.ba`) the whole
web client crashed at mount with:

```
OwlError: An error occured in the owl lifecycle
Caused by: Error: Element '<xpath expr="//t[@t-if='this.ui.isSmall']"
position="attributes">…' cannot be located in element tree
```

Every user gets a dead client — JS template inheritance is applied when the
`WebClient`/NavBar first renders, so one unresolvable xpath in any installed
module's template kills the UI globally.

## Root cause: core-version skew, not a module bug

The module is identical to upstream OCA/web 19.0, which targets **current
OCB 19.0**. But instances built on `pkgs.odoo19` run the **May-2026
oca-ocb snapshot** core, and OCB renamed the breakpoint accessor in
`web.NavBar.AppsMenu` *after* that snapshot:

| | `web.NavBar.AppsMenu` (`addons/web/static/src/webclient/navbar/navbar.xml`) |
|---|---|
| oca-ocb snapshot (May 2026, `pkgs.odoo19`) | `<t t-if="env.isSmall">` |
| current OCB 19.0 (and OCA/web 19.0 target) | `<t t-if="this.ui.isSmall">` |

The template *structure* is otherwise identical — only the expression text
changed, and upstream's xpath anchors on exactly that text.

## Fix (commit `3a08539`) — valid on BOTH cores

`static/src/components/apps_menu/apps_menu.xml`:

1. **Structural xpath instead of expression text**:
   `//t[@t-if='this.ui.isSmall']` → `//a[@t-ref='menuApps']/..`
   (the parent `<t t-if>` of the sidebar toggle `<a t-ref="menuApps">`;
   the JS inheritance engine uses real `document.evaluate` XPath, so the
   parent axis works). Resolves on both cores.
2. **Runtime expression** in the `//Dropdown` replacement:
   `<t t-if="this.ui.isSmall">` → `<t t-if="env.isSmall">`.
   The old core's NavBar component has **no `this.ui`** — even with the
   xpath fixed, rendering would throw. `env.isSmall` exists and is still
   used in current OCB, so it works on both.

Because both changes are valid against the current OCB too, the fix
**survives the fleet-proper 4.1 core refresh** (regenerating `pkgs.odoo19`
from live OCB 19.0) — no revert needed, though after 4.1 the module can be
re-synced to pristine upstream if preferred.

## Verification done

All template-inheritance xpaths in the module (~30 across `apps_menu.xml`,
`chatter.xml`, `command_palette/main.xml`, `file_viewer.xml`,
`form_buttons.xml`, `hotkey.xml`, `custom_favorite_item.xml`,
`menu_fuse_searchbar/searchbar.xml`) were machine-validated with an lxml
script (translating Odoo's `hasclass(...)` into standard XPath
`contains(concat(' ', @class, ' '), …)`) against **both** template trees:

* old core: `git archive 19.0 -- ./web/static/src` from
  `packages/oca-ocb-core/odoo-bringout-oca-ocb-web` (+ `…-ocb-mail` for
  `mail.Chatter`),
* new core: `tmp/oca_repos/OCA__OCB_19_0/addons`.

Result: the `t-if` anchor was the **only** mismatch; after the fix both
trees validate with 0 failures. The JS side needed no changes — the asset
bundle compiled fine on the instance (the failure was at owl render, which
proves all `@web/...` imports resolve on the old core).

## Deploying to an instance

Re-stage the module into the instance addons dir and regenerate assets
(`-u web_responsive`, or delete the `ir.attachment` asset bundles /
restart with `--dev=assets` once). A browser hard-reload is needed since
the broken bundle is cached under its hash URL.

## Lesson for other OCA 19.0 modules on the stale bundle

Until 4.1 refreshes `pkgs.odoo19`, ANY vendored OCA 19.0 module whose JS
template xpaths anchor on expression text is at risk of this exact
client-wide crash. When one appears ("cannot be located in element tree"
in the browser console): diff the target template between the two cores,
then re-anchor structurally with expressions valid on both.
