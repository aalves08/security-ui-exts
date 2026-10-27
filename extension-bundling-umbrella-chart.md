# Bundling multiple UI extensions into a single Helm chart

Investigation into whether `rancher/security-ui-exts` can ship its extensions as one
installable unit on the Rancher Dashboard Extensions screen, using a Helm umbrella
chart with subcharts.

- **Dashboard reference:** `rancher/dashboard@master` (read via local fork)
- **Backend reference:** `rancher/rancher@main`
- **Extension repo:** `rancher/security-ui-exts@main`

> ## ⚠️ Hard requirement: ONE CARD
>
> A bundle MUST appear on the Extensions screen as **exactly one card**. Not one card
> plus hidden members, not one card plus orphans. One. Any design below that produces
> more than one card for a single bundle is wrong, not a trade-off.
>
> **This is not achievable by chart packaging alone.** Cards come from three independent
> sources in `available()`, and all three must be handled — see
> [§3.7 Getting to exactly one card](#37-getting-to-exactly-one-card), which is the
> checklist to work against.

> **Verification status (updated 2026-10-27):** the chart-level smoke test
> ([step 1 of the verification plan](#9-verification-plan)) **has now been run** — see
> [§9.1 Smoke test results](#91-smoke-test-results-run-2026-10-27). It confirmed the
> umbrella mechanics and corrected three things in this document (§4, §5.3, §6.2).
> Nothing has yet been run against a live Rancher, so §2 (backend indifference) and the
> Extensions-screen failure modes in §3.1–3.3 remain source-reading only.
>
> Claims about *current* behaviour are traced to specific files and line numbers and are
> reliable. Claims marked **VERIFIED** were executed. Everything else is design reasoning.

---

## 1. How extension charts are built and served today

`.github/workflows/build-extension-charts.yml` in `rancher/dashboard` is a reusable
(`workflow_call`) workflow. `security-ui-exts` already calls it from its own
`release.yml` on release publish. The workflow itself is thin — the real work is
`yarn publish-pkgs`, which resolves to `shell/scripts/extension/publish` inside
`@rancher/shell`.

That script does, in order:

1. **Iterates `pkg/*/`.** One Helm chart per package directory. This is hardcoded —
   there is no grouping concept anywhere in the pipeline.
2. **Builds each package** via `shell/scripts/build-pkg.sh`. Output is
   `<pkg>-<version>.umd.min.js`, and critically the UMD **global variable name** is
   also `<pkg>-<version>` (`build-pkg.sh:35,70`).
3. **Copies the generic chart template** from
   `shell/scripts/extension/helm/charts/ui-plugin-server/`. This chart has exactly one
   template: a `UIPlugin` custom resource (`templates/cr.yaml`). No Deployment, no
   Service — the chart is a pointer, not a workload.
4. **Patches it** with `helm/scripts/patch` (rewrites `Chart.yaml` name / version /
   description / icon / keywords / annotations from the package's `package.json`) and
   `helmpatch` (sets `values.plugin.endpoint` to
   `https://raw.githubusercontent.com/<repo>/gh-pages/extensions/<pkg>/<version>`).
5. **Packages and indexes** — `helm package` into `assets/<pkg>/<pkg>-<version>.tgz`,
   then `helm repo index` into `index.yaml`.
6. **Commits all of it to `gh-pages`.** That branch simultaneously *is* the Helm
   repository (`index.yaml` + `assets/`) and the static file server for the extension
   JavaScript (`extensions/<pkg>/<version>/plugin/...`).

The Dashboard then treats any chart carrying `catalog.cattle.io/ui-component: plugins`
as an extension (`shell/config/uiplugins.js:14-17`).

### Release tagging

`shell/scripts/extension/parse-tag-name` enforces that the GitHub release tag matches
`<pkg-dir>-<version>` for some directory under `pkg/`. A tag that matches nothing
cancels the run. `publish` independently skips any package whose `<pkg>-<version>`
doesn't equal the tag. So today: **one GitHub release per extension**.

### Current state of `security-ui-exts`

| Package | Version | Notes |
|---|---|---|
| `common` | 0.0.0 | `private: true` — shared code, not an extension |
| `network` | 0.1.0 | no display-name/icon set |
| `runtime-enforcer` | 0.1.0 | `prime-only: true` |
| `vulnerability-scanner` | 0.9.1 | `prime-only: true` |

Root `package.json` is at `0.9.1`, mirroring `vulnerability-scanner`. That only matters
for the catalog-image flow today, but it becomes load-bearing if we add bundling — see
[open question 2](#8-open-questions--decisions-needed).

> **Incidental bug, unrelated to bundling:** `publish` iterates `pkg/*/` without
> checking `private: true`. `common` is only skipped today because the release-tag
> filter excludes it. A `head` build (`-t head`) would try to publish `common-0.0.0`
> as an extension chart. Worth a one-line guard in `publish` regardless of what we
> decide here.

---

## 2. The backend does not care — confirmed

This was the main open risk and it is resolved. The Rancher backend has **no Helm
awareness whatsoever** in the plugin path.

`pkg/controllers/dashboard/plugin/controller.go` — `OnPluginChange` lists every
`UIPlugin` CR in the namespace with no selector and rebuilds a flat index:

```go
cachedPlugins, err := h.pluginCache.List(h.systemNamespace, labels.Everything())
...
err = Index.Generate(cachedPlugins)
```

`pkg/controllers/dashboard/plugin/index.go` — keyed purely by the plugin's own name:

```go
for _, plugin := range cachedPlugins {
    entry := plugin.Spec.Plugin
    s.Entries[entry.Name] = &UIPlugin{
        UIPluginEntry: entry,
        CacheState:    plugin.Status.CacheState,
        Ready:         plugin.Status.Ready,
    }
}
```

`pkg/api/steve/catalog/plugin.go` serves `/v1/uiplugins/{name}/{version}/{rest...}`
straight out of that index.

No owner references, no release names, no chart provenance — the controller never asks
where a CR came from. Each CR gets its own filesystem cache sync, its own
`Ready`/`CacheState` status and its own retry backoff.

**Four `UIPlugin` CRs created by one Helm release are indistinguishable, to the backend,
from four CRs created by four releases.** Runtime loading is equally unaffected:
`shell/plugins/plugin.js:72-80` iterates `/v1/uiplugins` entries independently.

### Correction to a common assumption

Subcharts are **not** "installed one at a time". A Helm umbrella chart is a **single
release** — parent and all subcharts render into one manifest set applied together.
`helm list` shows one row. Consequences that need to be accepted up front:

- **Uninstall is all-or-nothing.** You cannot `helm uninstall` one member. Removing one
  means `helm upgrade` with that subchart's `condition` flag flipped false (workable, if
  we wire `condition:` into the `dependencies:` entries).
- **Upgrade is all-or-nothing.** One user-facing version for the set.

For the stated goal — "all of them installed in one go" — this is arguably the desired
behaviour, not a limitation. But it is a one-way door per release, so it should be a
deliberate choice.

---

## 3. What actually breaks: the Extensions management screen

Everything in `shell/pages/c/_cluster/uiplugins/index.vue` assumes
**one chart ⟷ one `UIPlugin` CR ⟷ one Helm release, all sharing a name.**

### 3.1 Install state never resolves

`wirePluginCRToChart` (`index.vue:966-1031`) finds the Helm App by CR name and then the
chart by that same name:

```js
const app = this.apps.find((a) => a.metadata.name === pluginCR.name && ...);
const originalRepoName = app?.spec?.chart?.metadata?.annotations?.[CATALOG_ANNOTATIONS.SOURCE_REPO_NAME] || ...;
if (originalRepoName) {
  chart = all.find((c) => c.name === pluginCR.name && c.chart?.repoName === originalRepoName);
}
```

With a bundle the App is `security-ui-exts` and the CRs are `network`,
`runtime-enforcer`, `vulnerability-scanner`. Nothing matches, so:

- the `security-ui-exts` card stays in **Available** with an Install button, forever;
- three orphan cards land in **Installed** via the `else` branch at `index.vue:1013` —
  no icon, no description (`pluginCR.description` reads `spec.plugin.description`, which
  the chart template never sets), `versions: []`, no upgrade path.

### 3.2 Uninstall silently does nothing

`shell/dialog/UninstallExtensionDialog.vue:80-82`:

```js
const pluginApp = apps.find((app) => {
  return app.namespace === UI_PLUGIN_NAMESPACE && app.name === plugin.name;
});
if (pluginApp) { /* ... remove ... */ }
```

For an orphan card `plugin.name` is `network`; the App is `security-ui-exts`. No match,
`if` is false, dialog closes, nothing happens, no error. The card stays.

There *is* a working path — Apps → Installed Apps → delete the release removes all CRs
cleanly — but nobody will find it.

### 3.3 Duplicate cards from the runtime plugin list

`available()` (`index.vue:272-280`) separately pushes a card for every *loaded* plugin
not already present:

```js
this.uiplugins.forEach((p) => {
  if (!all.find((c) => c.name === p.name)) {
    const item = this.buildLoadedPluginItem(p);
    if (item) all.push(item);
  }
});
```

Each bundle member produces a card here too, independent of 3.1.

> **Important correction — where `this.uiplugins[].metadata` actually comes from.**
> An earlier draft of this document said `this.uiplugins` comes from `/v1/uiplugins` and
> therefore carries `spec.plugin.metadata`. **That is wrong**, and it invalidates the
> original fix proposed in §5.3. The real chain:
>
> 1. `/v1/uiplugins` is used only to *locate and load* the JavaScript —
>    `loadPluginAsync` reads `metadata.direct` and `metadata.main` off the index entry to
>    build the URL (`extension-manager-impl.js:88-108`).
> 2. On script load, a **brand new object** is created —
>    `const plugin = new Plugin(id)` (`extension-manager-impl.js:141`). Nothing from the
>    CR is copied onto it.
> 3. The extension's own entrypoint is then invoked with that object, and does:
>    ```ts
>    // pkg/vulnerability-scanner/index.ts:24
>    plugin.metadata = require('./package.json');
>    ```
>    `shell/core/plugin.ts:83` documents the field as *"Plugin metadata (plugin
>    package.json)"*.
> 4. That object is pushed to the store (`extension-manager-impl.js:160`), and
>    `mapGetters({ uiplugins: 'uiplugins/plugins' })` is what `available()` iterates.
>
> **So `this.uiplugins[].metadata` is the extension's `pkg/<name>/package.json`, baked
> into the UMD bundle at build time. Helm values cannot influence it at all.** This is
> also why `buildLoadedPluginItem` reads the nested `plugin.metadata.rancher.annotations`
> shape — it is reading package.json, not the CR.
>
> The two lists therefore carry genuinely different data:
>
> | List | Source | Bundle marker reachable? |
> |---|---|---|
> | `this.plugins` | `UIPlugin` CRs via `management/findAll` | **Yes** — `pluginMetadata` getter → `spec.plugin.metadata` |
> | `this.uiplugins` | `new Plugin()` + the extension's own `package.json` | **No** — unless injected into package.json pre-build |

### 3.4 The naming constraint that breaks bundles outright — **VERIFIED**

Not a Dashboard bug, but fatal if missed. The loader
(`shell/core/extension-manager-impl.js:88-108`) builds:

```js
const id  = `${ name }-${ version }`;
const url = `${ UI_PLUGIN_BASE_URL }/${ name }/${ version }/plugin/${ main }`;  // main defaults to `${id}.umd.min.js`
// ...later: window[id].default must be a function
```

The default `fullname` helper is `<release>-<chartname>`, which under a bundle yields
`security-ui-exts-network`. That URL 404s and the UMD global doesn't exist. **Each
subchart must set `fullnameOverride` to its package name.**

Confirmed by rendering the umbrella with the override removed:

```
$ helm template security-ui-exts <bundle> -n cattle-ui-plugin-system \
    --set vulnerability-scanner.fullnameOverride=null
  name: security-ui-exts-vulnerability-scanner     # metadata.name
    name: security-ui-exts-vulnerability-scanner   # spec.plugin.name
```

With `fullnameOverride` set, both collapse back to `vulnerability-scanner`. Note the
member does **not** need to carry its own version explicitly — see the correction in
[§6.2](#62-vendor-subcharts-directly--do-not-use-remote-dependencies).

Related: keep `metadata.name` identical to `spec.plugin.name`. `Index.Generate` keys by
`spec.plugin.name`, but `Index.Ready()` / `Index.CacheState()` look up
`s.Entries[plugin.Name]` — the CR's `metadata.name`. Diverge them and those status
writes silently no-op, leaving the entry at `CacheState: Pending`, which makes the
handler proxy to GitHub instead of serving from the FS cache (breaks airgap until the
next resync). The stock template uses the same helper for both, so this is free unless
hand-rolled.

### 3.5 Collision with an already-installed standalone extension

If a user has `vulnerability-scanner` installed standalone and then installs the bundle,
both produce a CR named `vulnerability-scanner` in `cattle-ui-plugin-system`. Helm
rejects this with an ownership error (`annotation validation error: key
meta.helm.sh/release-name must equal ...`) — a hard failure, not silent corruption, which
is the better outcome.

But the Dashboard's existing guard for this
(`showInstallDialog` → `UninstallExistingExtensionDialog`, `index.vue:498-523`) matches
on `p.name === plugin.name`. The bundle card's name is `security-ui-exts`, so the guard
never fires and the user gets a raw Helm error instead of the "uninstall the existing one
first" dialog.

### 3.6 Summary

| Capability | Works with an umbrella chart today? |
|---|---|
| Helm installs all members in one action | **Yes** |
| Backend indexes and serves all members | **Yes** (confirmed in source) |
| Extensions screen → Install button | **Yes** |
| Extensions screen → install state afterwards | **No** — see 3.1, 3.3 |
| Extensions screen → Uninstall button | **No** — silent no-op, see 3.2 |
| Extensions screen → Upgrade | **No** — no version list on orphan cards |
| Apps screen → delete release | **Yes** — removes everything cleanly |
| Guard against standalone/bundle conflict | **No** — see 3.5 |

So: *install works, uninstall works only if you know to use the Apps screen, and the
Extensions screen misreports state.* Acceptable for a controlled demo; not acceptable for
something customers operate.

### 3.7 Getting to exactly one card

Restating the hard requirement from the top of this document, as a checklist. Installing
a two-member bundle with no changes anywhere produces **up to five cards**. They come
from three independent sources inside `available()`
(`pages/c/_cluster/uiplugins/index.vue:265-283`), and **each has to be closed
separately**:

| # | Source | Cards it adds | What closes it |
|---|---|---|---|
| A | `this.charts` — catalog entries from `index.yaml`, filtered by `isUIPlugin` and `!hidden` (`index.vue:266-268`) | 1 per member chart published standalone | Don't publish member charts standalone, **or** publish them with `catalog.cattle.io/hidden: "true"` |
| B | `this.uiplugins` — one card per *loaded* extension whose name isn't already in the list (`index.vue:272-280`) | 1 per member, **always** | The §5.3 cross-reference fix. **Dashboard change — unavoidable.** |
| C | `wirePluginCRToChart` else-branch — one orphan card per `UIPlugin` CR that can't be matched to a chart (`index.vue:1013`) | 1 per member, **always** | The §5.2 bundle branch. **Dashboard change — unavoidable.** |
| — | The bundle's own catalog entry | 1 — **the one you want** | — |

**The critical point: A is the only one you can fix from the extension repo.** B and C
fire off the loaded plugins and the CRs, which exist no matter what is or isn't in the
Helm index. So:

- Removing the standalone charts from `index.yaml` **does not** give you one card. It
  gives you three instead of five.
- Marking them `hidden: true` **does not** either. Same three.
- **There is no packaging-only path to one card.** §5.2 and §5.3 are mandatory, not
  nice-to-have. That is the whole reason this needs a `rancher/dashboard` change.

For the POC, the cleanest way to demonstrate the requirement is met: publish the bundle
chart only (no standalone member entries in `assets/`), apply §5.2 + §5.3 + §5.4, and
confirm the Extensions screen shows a single `SUSE Security` card that installs,
reports installed state, and uninstalls both members.

---

## 4. Design decision that makes the fix tractable

The obvious fix is to read Helm's implicit `meta.helm.sh/release-name` annotation off the
CR. It works and has the nice property of applying retroactively to charts already
published.

**I recommend against it as the primary mechanism**, for one decisive reason: that
annotation lives on the CR's `metadata`, which is visible to `this.plugins` (full CRs
fetched via `management/findAll` on `UI_PLUGIN`) but not where problem 3.3 lives.

**Instead, stamp the bundle name into `spec.plugin.metadata`.** That field is already
part of `UIPluginEntry`, is served verbatim by the backend index, and is exposed on the
CR model by the existing `pluginMetadata` getter
(`models/catalog.cattle.io.uiplugin.js:26`):

```yaml
# rendered into each member's UIPlugin CR
spec:
  plugin:
    name: vulnerability-scanner
    version: 0.9.1
    metadata:
      catalog.cattle.io/ui-extensions-bundle: security-ui-exts
```

Bundles don't exist yet, so there's nothing to be retroactive *for* — the usual argument
for the Helm annotation doesn't apply here.

Keep reading `meta.helm.sh/release-name` as a secondary fallback in
`wirePluginCRToChart` if we want belt-and-braces, but the metadata key is the contract.

**VERIFIED** — the marker merges cleanly. Setting
`vulnerability-scanner.plugin.metadata.catalog.cattle.io/ui-extensions-bundle` in the
parent `values.yaml` *deep-merges* into the subchart's own `plugin.metadata` rather than
replacing it, so the display-name and compatibility annotations written by
`helm/scripts/patch` survive alongside it:

```yaml
    metadata:
      catalog.cattle.io/display-name: "SUSE Security Vulnerability Scanner"
      catalog.cattle.io/kube-version: ">= v1.16.0-0"
      catalog.cattle.io/prime-only: "true"
      catalog.cattle.io/rancher-version: ">= 2.11.0-0"
      catalog.cattle.io/ui-extensions-bundle: "security-ui-exts"      # <-- injected by parent
      catalog.cattle.io/ui-extensions-version: ">= 3.0.0 < 4.0.0"
```

### 4.1 Correction: this is *not* a single channel

The original version of this section claimed `spec.plugin.metadata` "reaches both
consumers through one channel". **It does not** — see the correction box in
[§3.3](#33-duplicate-cards-from-the-runtime-plugin-list). It reaches `this.plugins`
only. `this.uiplugins` is populated from the extension's own `package.json`, compiled
into the UMD bundle, and no Helm value can reach it.

So the marker is still the right contract for §5.2 and §5.4 (both operate on CRs), but
§5.3 needs a different mechanism. Two options:

**Option A — cross-reference the two lists in `available()` (recommended).**
Build a set of bundled member names from `this.plugins` (where the marker *is*
available) and use it to filter the `this.uiplugins` loop. Roughly four lines, and
`this.plugins` is already iterated two lines further down, so nothing new is fetched.

**Option B — inject the bundle name into `pkg/<name>/package.json` before `build-pkg`.**
Add e.g. `rancher.bundle: "security-ui-exts"`, so it is baked into the UMD and
`buildLoadedPluginItem` can read it directly with no cross-referencing. This is the
natural reading of *"the bundle definition should drive the members"*, and it is the
only option that makes a loaded extension self-aware of its suite.

**Recommendation: A.** Option B makes the member's compiled JavaScript bundle-specific,
which means the same build can no longer be published both standalone and bundled —
that kills [open question 1](#8-open-questions--decisions-needed), which is otherwise
the recommended answer. A keeps exactly one artifact per member. Revisit B only if
something at runtime genuinely needs to know its own suite membership.

---

## 5. Dashboard changes

All paths relative to `shell/`.

### 5.1 `config/uiplugins.js` — new constant

```js
// Plugin metadata key identifying the bundle (umbrella chart release) a member belongs to
export const UI_PLUGIN_METADATA = {
  BUNDLE: 'catalog.cattle.io/ui-extensions-bundle',
};
```

### 5.2 `pages/c/_cluster/uiplugins/index.vue` — `wirePluginCRToChart`

Branch on bundle membership. A member CR must wire onto the **bundle's** chart card and
must not create an orphan card.

```js
wirePluginCRToChart(pluginCR, all) {
  const bundleName = pluginCR.pluginMetadata?.[UI_PLUGIN_METADATA.BUNDLE];

  if (bundleName) {
    return this.wireBundledPluginCRToChart(pluginCR, bundleName, all);
  }

  // ...existing single-extension logic, unchanged...
}

wireBundledPluginCRToChart(pluginCR, bundleName, all) {
  const app = this.apps.find((a) => a.metadata.name === bundleName && a.metadata.namespace === UI_PLUGIN_NAMESPACE);
  const originalRepoName = app?.spec?.chart?.metadata?.annotations?.[CATALOG_ANNOTATIONS.SOURCE_REPO_NAME] ||
                           app?.metadata?.labels?.[CATALOG_ANNOTATIONS.CLUSTER_REPO_NAME];

  let chart = originalRepoName
    ? all.find((c) => c.name === bundleName && c.chart?.repoName === originalRepoName)
    : null;

  chart = chart || all.find((c) => c.name === bundleName);

  if (!chart) {
    // Bundle chart isn't in the catalog (repo removed, or installed from a repo we can't
    // see). Fall back to a single aggregate card rather than one card per member.
    return this.pushOrphanBundleCard(pluginCR, bundleName, app, all);
  }

  chart.installed = true;
  chart.isBundle  = true;
  chart.bundleMembers = [...(chart.bundleMembers || []), pluginCR];

  // The bundle's version is the chart/release version, NOT any member's version.
  chart.installedVersion = app?.spec?.chart?.metadata?.version || chart.installedVersion;
  chart.installing = this.installing[chart.id];

  const latest = chart.installableVersions?.[0];

  if (latest && chart.installedVersion && chart.installedVersion !== latest.version) {
    chart.upgrade = getPluginChartVersionLabel(latest);
  }
}
```

Note `chart.bundleMembers` accumulates across calls, since `available()` invokes this
once per CR. `available()` is a computed rebuilt from scratch each time, so there's no
stale-accumulation risk — but it does mean `wireBundledPluginCRToChart` must be
idempotent with respect to `installed`/`installedVersion`, which it is.

### 5.3 `pages/c/_cluster/uiplugins/index.vue` — `available()`

Suppress per-member cards from the runtime plugin list (problem 3.3).

> **This section was rewritten.** The original proposed
> `if (p.metadata?.[UI_PLUGIN_METADATA.BUNDLE]) return;`. That **does not work** —
> `p.metadata` is the extension's `package.json`, not the CR's `spec.plugin.metadata`.
> See the correction box in [§3.3](#33-duplicate-cards-from-the-runtime-plugin-list)
> and [§4.1](#41-correction-this-is-not-a-single-channel).

Derive the member-name set from `this.plugins` (the CRs, where the marker *is*
reachable) and filter the `uiplugins` loop with it:

```js
// Names of extensions that are members of an installed bundle. Sourced from the CRs,
// because this.uiplugins entries carry the extension's own package.json, not CR data.
const bundledMemberNames = new Set(
  this.plugins
    .filter((cr) => cr.pluginMetadata?.[UI_PLUGIN_METADATA.BUNDLE])
    .map((cr) => cr.name)
);

this.uiplugins.forEach((p) => {
  // Members of a bundle are represented by the bundle's own card
  if (bundledMemberNames.has(p.name)) {
    return;
  }

  if (!all.find((c) => c.name === p.name)) {
    const item = this.buildLoadedPluginItem(p);

    if (item) {
      all.push(item);
    }
  }
});
```

`this.plugins` is already iterated immediately below this block, so there is no extra
fetch and no ordering problem.

#### Latent bug worth fixing while we're here

`buildLoadedPluginItem` reads `plugin.metadata?.rancher.annotations?.[...]` for the
display name and the prime-only / experimental / certified flags. For a chart-installed
extension, `package.json` does contain `rancher.annotations`, so this path works — but
it means a loaded extension's card is labelled from **package.json**, while a
catalog-resolved card is labelled from the **chart annotations** via
`mapChartToPluginItem`. Those two can drift, because `helm/scripts/patch` writes the
chart's copy as flat keys onto `values.plugin.metadata`. Independent of bundling, but it
is the reason orphan cards in 3.1 look so bare.

### 5.4 `dialog/UninstallExtensionDialog.vue`

Resolve the App by bundle name when present, and tell the user what they're actually
removing.

```js
const releaseName = plugin.uiplugin?.pluginMetadata?.[UI_PLUGIN_METADATA.BUNDLE] ||
                    plugin.chart?.chartName ||
                    plugin.name;

const pluginApp = apps.find((app) => app.namespace === UI_PLUGIN_NAMESPACE && app.name === releaseName);
```

Add an `else` branch that growls an error when no App is found — the current silent
no-op is a bug for non-bundle cases too (it's how 3.2 manifests, but it would equally
hide any other resolution failure).

Template change: when `plugin.isBundle`, list `plugin.bundleMembers` in the confirmation
so "Uninstall SUSE Security" makes it obvious that three extensions are going away.
Needs a new i18n key, e.g. `plugins.uninstall.bundlePrompt`.

### 5.5 `dialog/InstallExtensionDialog.vue`

The `exists` check already uses `plugin.chart.chartName`, which for the bundle card is
the bundle chart name — correct as-is, no change needed.

One real change: `chartVersionLoadsWithoutAuth()` reads
`this.chartVersionInfo?.values?.plugin?.noAuth`. Under an umbrella chart, `plugin.noAuth`
lives under each subchart key, not at the root. Either hoist a root-level
`plugin.noAuth` in the parent `values.yaml` (simplest — see 6.2) or teach this getter to
scan subchart values. Recommend hoisting; keeps the Dashboard dumb.

### 5.6 `pages/c/_cluster/uiplugins/index.vue` — conflict guard (3.5)

Extend the existing check so installing a bundle detects already-installed members:

```js
if (action === 'install') {
  const conflicting = plugin.isBundleChart
    ? this.available.filter((p) => p.installed && plugin.bundleMemberNames?.includes(p.name))
    : [this.available.find((p) => p.name === plugin.name && p.installed)].filter(Boolean);
  // ...prompt UninstallExistingExtensionDialog with the conflicting set...
}
```

`bundleMemberNames` has to come from the catalog entry (pre-install) rather than from
CRs. Two ways to get it, and **VERIFIED** that both are present in `index.yaml`:

1. **An explicit annotation.** Add
   `catalog.cattle.io/ui-extensions-bundle-members: "runtime-enforcer,vulnerability-scanner"`
   to the bundle chart's `Chart.yaml` in the build step (see 6.1).
2. **Helm's own `dependencies` block**, which `helm repo index` copies into the index
   entry verbatim — no extra annotation needed:

   ```json
   "dependencies": [
     {"name":"vulnerability-scanner","version":"0.9.1","condition":"vulnerability-scanner.enabled","repository":""},
     {"name":"runtime-enforcer","version":"0.1.0","condition":"runtime-enforcer.enabled","repository":""}
   ]
   ```

   This is strictly better information — it carries each member's *version* and its
   enable condition, not just a name, and it cannot drift from the actual chart
   contents the way a hand-maintained annotation can.

   Caveat: it still has to be confirmed that `dependencies` survives onto the chart
   object the Dashboard sees. `chart.versions[]` entries look like raw index entries
   (`store/catalog.ts`), so it very likely does, but this was **not** verified.

**Recommendation:** read `dependencies` with the annotation as a fallback. Keep writing
the annotation anyway — it is cheap, and it is the only one of the two that is legible
to someone reading `Chart.yaml`.

This also lets the Available card list its contents before installation, which is good
UX.

`UninstallExistingExtensionDialog` currently takes a single `installedPlugin` prop — it
needs to accept a list.

### 5.7 Card presentation (nice-to-have, not blocking)

`getSubHeaderItems` / `getFooterItems` / `PluginInfoPanel.vue` could show
"3 extensions" and enumerate them. Purely additive.

---

## 6. Chart generation changes

### 6.1 New parent chart template

Add `shell/scripts/extension/helm/charts/ui-plugin-bundle/` — `Chart.yaml` plus
`values.yaml`, and **no templates of its own**. All rendered content comes from
subcharts.

```yaml
# Chart.yaml (template; name/version/annotations patched at build time)
apiVersion: v2
type: application
name: extension-bundle
version: 0.0.0
appVersion: "0.0.0"
annotations:
  catalog.cattle.io/certified: rancher
  catalog.cattle.io/namespace: cattle-ui-plugin-system
  catalog.cattle.io/os: linux
  catalog.cattle.io/permits-os: linux, windows
  catalog.cattle.io/scope: management
  catalog.cattle.io/ui-component: plugins
```

### 6.2 Vendor subcharts directly — do not use remote `dependencies:`

Rancher's helm operation installs from the packaged `.tgz` served by the ClusterRepo, and
Helm will not resolve missing dependencies at install time — it errors with
*"found in Chart.yaml, but missing in charts/ directory"*. Copy the already-generated
per-package charts straight into the parent's `charts/` directory. No
`helm dependency update`, no network at install time, airgap-safe.

**VERIFIED** — `helm package` accepts vendored subcharts with no `charts/*.tgz` and no
dependency resolution step, and the resulting archive is self-contained:

```
security-ui-exts/Chart.yaml
security-ui-exts/values.yaml
security-ui-exts/charts/runtime-enforcer/{Chart.yaml,values.yaml,templates/*}
security-ui-exts/charts/vulnerability-scanner/{Chart.yaml,values.yaml,templates/*}
```

### Correction: `versionOverride` is not needed

The original draft specified a per-member `plugin.versionOverride`. **It is redundant.**
Each subchart gets its *own* `.Chart` context, and `helm/scripts/patch` has already set
that subchart's `appVersion` to the package version. So the existing
`default .Chart.AppVersion .Values.plugin.versionOverride` in `cr.yaml` resolves
correctly with no override at all — **VERIFIED**, rendering `0.9.1` and `0.1.0` for the
two members off a bundle whose own version is `1.0.0`.

The same applies to `endpoint` / `compressedEndpoint` / `metadata`: `helmpatch` already
wrote correct per-member values into each subchart's own `values.yaml`, and vendoring
copies them in wholesale. **The parent only needs to add what it alone knows**, which is
`fullnameOverride` and the bundle marker:

```yaml
# hoisted for InstallExtensionDialog (5.5)
plugin:
  noAuth: false

vulnerability-scanner:
  enabled: true
  fullnameOverride: vulnerability-scanner        # REQUIRED — see 3.4
  plugin:
    metadata:
      catalog.cattle.io/ui-extensions-bundle: security-ui-exts

runtime-enforcer:
  enabled: true
  fullnameOverride: runtime-enforcer
  plugin:
    metadata:
      catalog.cattle.io/ui-extensions-bundle: security-ui-exts
```

That is the whole parent `values.yaml`. Everything else is inherited.

**VERIFIED** — per-member opt-out works: with `condition: <pkg>.enabled` on each
`dependencies:` entry and `enabled: true` defaulted in the parent values,
`--set runtime-enforcer.enabled=false` renders one CR instead of two.

### 6.3 Compatibility annotations

`catalog.cattle.io/rancher-version`, `ui-extensions-version` and `kube-version` gate
whether the card is installable (`isSupportedChartVersion`). For a bundle these must be
the **intersection** of all members' ranges. Intersecting semver ranges in bash is not
worth attempting — require the bundle config to declare its own ranges explicitly, and
have the build **fail** if a member's declared range is not a subset. Simpler and the
error is actionable.

Also note `runtime-enforcer` and `vulnerability-scanner` are `prime-only: true` while
`network` is not. The bundle must be `prime-only: true` (most restrictive wins), which
means `network` effectively becomes Prime-only when bundled. Flag this to product.

> `prime-only` is a **display label only** — `store/catalog.ts:660` pushes it onto
> `tags`, and `index.vue:795` renders it. There is no install gate anywhere. So it
> affects presentation and messaging, not whether the bundle can be installed.

### 6.4 Should the bundle's annotations propagate down to the members?

Short answer: **yes, do it — but understand that it buys consistency, not function.**
Here is where each copy of an annotation actually lands, all **VERIFIED** by inspecting
the rendered output and the generated `index.yaml`:

| Location | Reaches the catalog? | Reaches runtime? | Who reads it |
|---|---|---|---|
| **Parent `Chart.yaml` annotations** | **Yes** — copied verbatim into `index.yaml` | no | `isSupportedChartVersion`, prime-only/certified tags, display name |
| Subchart `Chart.yaml` annotations | **No** | no | nobody — Helm uses only `name`/`version`/`condition` from a vendored subchart |
| Subchart `values.plugin.metadata` | no | **Yes** — into `spec.plugin.metadata`, served by `/v1/uiplugins` | `loadPluginAsync` (`direct`, `main`), the `pluginMetadata` getter, and our new bundle branch |

The decisive detail: **every compatibility and certification check in the Dashboard
reads the `chart` object, never the CR.** Confirmed by grepping every call site of
`uiPluginHasAnnotation` / `uiPluginAnnotation` — they all take `chart` or
`plugin?.chart`. So `rancher-version`, `ui-extensions-version`, `kube-version`,
`prime-only` and `certified` sitting in a member's `spec.plugin.metadata` are **inert**.
Only the parent's `Chart.yaml` gates anything.

That means propagation is not load-bearing for the bundle to work. It is still worth
doing, for three reasons:

1. **The vendored subchart stays byte-identical to the standalone chart.** If members
   are also published à-la-carte ([open question 1](#8-open-questions--decisions-needed)),
   you want one generator and one result, not two subtly different charts.
2. **`spec.plugin.metadata` is the only self-description that survives onto the
   cluster.** Anyone debugging a `UIPlugin` CR, or any future tooling reading
   `/v1/uiplugins`, sees the suite's declared compatibility rather than nothing.
3. **It makes drift detectable.** Once the bundle definition is the single source, the
   build can diff each member's `pkg/*/package.json` ranges against the bundle's and
   fail loudly, which is exactly the subset check proposed above.

**Recommended propagation rule for the build script:**

- Parent `Chart.yaml` annotations ← **the bundle definition only** (§7.2). This is the
  gate; it must be deliberate, never computed from members.
- Each member's `values.plugin.metadata` ← the member's own annotations (already done by
  `patch`), **plus** the bundle's annotations overlaid on top, **plus** the
  `ui-extensions-bundle` marker. Deep-merge semantics make the overlay a one-liner in
  the parent `values.yaml`, and it is already **VERIFIED** working for the marker.
- Subchart `Chart.yaml` ← leave exactly as `patch` generated it. Overwriting it has no
  effect on anything and only makes the vendored copy diverge from the standalone one.
- **Fail the build** if any member declares a compatibility range that is not a subset
  of the bundle's. Do not attempt to intersect ranges.

One caveat on the overlay: where the bundle and the member disagree, the bundle wins,
and `display-name` is the annotation where you almost certainly *don't* want that — the
bundle's display name is "SUSE Security", and stamping that onto every member's CR would
make all of them identical. **Exclude `catalog.cattle.io/display-name` from the overlay**
(and anything else member-identifying, e.g. `icon`).

> For the two-package POC this is all moot in practice: `vulnerability-scanner` and
> `runtime-enforcer` already declare **identical** `rancher-version`,
> `ui-extensions-version` and `kube-version` ranges and are **both** already
> `prime-only: true`. The intersection is a copy-paste and the subset check passes
> trivially. Dropping `network` also makes
> [open question 3](#8-open-questions--decisions-needed) disappear entirely.

---

## 7. Build pipeline changes

### 7.1 `shell/scripts/extension/publish` — bundle mode

New flag, e.g. `-B` to enable bundle mode. After the existing per-package loop (members
must be built first, since the bundle vendors their charts):

1. Read the bundle definition (see 7.2).
2. Create `charts/<bundle>/<bundle-version>/` from the `ui-plugin-bundle` template.
3. For each member, copy the generated `charts/<pkg>/<pkg-version>/` into
   `charts/<bundle>/<bundle-version>/charts/<pkg>/`.
4. Write the parent `values.yaml` and `dependencies:` block as in 6.2.
5. Patch parent `Chart.yaml` — name, version, description, icon, annotations, plus
   `catalog.cattle.io/ui-extensions-bundle-members`.
6. `helm package` into `assets/<bundle>/`.
7. Let the existing `helm repo index` step pick it up — no change needed there.

The existing "check existing charts for updates" loop at the top iterates `assets/*/`
and only md5-compares `Chart.yaml`, so the bundle slots in without modification.

`chart-releaser` runs with `charts_dir: ./charts/*`; the bundle sits at
`charts/<bundle>/<version>` like any other chart, so no workflow change there. **Confirm
in the prototype that `chart-releaser` handles vendored subcharts** — I haven't verified
this.

### 7.2 Where the bundle is defined

You suggested passing the member list as a workflow input. One blocker:
**`workflow_call` inputs only support `string`, `number` and `boolean` — there is no
array type.** `bundle-arr: ['a','b']` isn't expressible; it'd have to be a
comma-separated string.

More importantly, a bundle needs a version, a display name, a description, an icon and
compatibility ranges — all of which must be versioned alongside the code and must match
the release tag. Workflow inputs are the wrong home for that.

**Recommended: hybrid.** A boolean workflow input toggles bundle mode; the content lives
in the repo.

```jsonc
// security-ui-exts/package.json (root)
{
  "rancher": {
    "bundles": [{
      "name": "security-ui-exts",
      "version": "1.0.0",
      "displayName": "SUSE Security",
      "description": "Vulnerability scanning, runtime enforcement and network policy for Rancher",
      "icon": "https://raw.githubusercontent.com/rancher/security-ui-exts/main/pkg/.../logo.svg",
      "packages": ["vulnerability-scanner", "runtime-enforcer", "network"],
      "annotations": {
        "catalog.cattle.io/rancher-version": ">= 2.11.0-0",
        "catalog.cattle.io/ui-extensions-version": ">= 3.0.0 < 4.0.0",
        "catalog.cattle.io/kube-version": ">= v1.16.0-0",
        "catalog.cattle.io/prime-only": "true"
      }
    }]
  }
}
```

This mirrors how `rancher.annotations` already works per package, so it's a familiar
shape for extension authors. An array allows more than one bundle per repo later.

### 7.3 `shell/scripts/extension/parse-tag-name`

Add a `bundle` workflow type. Tag convention `<bundle-name>-<bundle-version>`, validated
against `rancher.bundles[].{name,version}` rather than against `pkg/` directories.

Note the collision risk: if a bundle is named the same as a package, tags become
ambiguous. Have the build reject a bundle name that matches any `pkg/` directory.

### 7.4 `.github/workflows/build-extension-charts.yml`

Two new optional inputs:

```yaml
    inputs:
      # ...existing...
      bundle:
        description: 'Build bundled (umbrella) charts defined in rancher.bundles'
        required: false
        type: boolean
        default: false
      bundle_name:
        description: 'Build only this bundle (defaults to all defined bundles)'
        required: false
        type: string
```

Wire into the two existing steps:

```yaml
      - name: Parse Extension Name
        run: |
          yarn parse-tag-name ${{ inputs.tagged_release }} ${{ github.run_id }} \
            "${{ inputs.bundle == true && 'bundle' || 'charts' }}"

      - name: Run build script
        run: |
          publish="yarn publish-pkgs -s ${{ github.repository }} -b ${{ inputs.target_branch }}"
          # ...existing flags...
          if [[ "${{ inputs.bundle }}" == "true" ]]; then
            publish="$publish -B"
            if [[ -n "${{ inputs.bundle_name }}" ]]; then
              publish="$publish -N ${{ inputs.bundle_name }}"
            fi
          fi
          $publish
```

### 7.5 `security-ui-exts/.github/workflows/release.yml`

```yaml
jobs:
  build-extension-charts:
    uses: rancher/dashboard/.github/workflows/build-extension-charts.yml@<new-sha>
    permissions:
      contents: write
      pull-requests: write
    with:
      target_branch: gh-pages
      tagged_release: ${{ github.ref_name }}
      bundle: true
```

Releases then tag `security-ui-exts-1.0.0` instead of (or alongside) the per-extension
tags. The workflow needs to handle both tag shapes if we keep publishing standalone
charts — easiest is to let `parse-tag-name` accept either and set bundle mode from the
tag shape rather than the input, with the input as an override.

---

## 8. Open questions / decisions needed

1. ~~**Publish members standalone as well as bundled?**~~ **DECIDED: no, not for the
   POC.** Publish the bundle chart only. The one-card requirement
   ([§3.7](#37-getting-to-exactly-one-card)) makes à-la-carte installs a separate
   feature, not a default, and shipping both doubles the conflict surface in 3.5 for no
   POC benefit.

   If à-la-carte is wanted later, the mechanism is `catalog.cattle.io/hidden: "true"` on
   the member charts — `available()` already filters on it (`index.vue:268`), so they
   stay resolvable and installable by name without showing as cards. **But note this
   only closes source A in §3.7**; it does nothing about B and C, so it is not a
   shortcut to the one-card requirement.

2. **Where does the bundle version come from?**
   Root `package.json` is currently `0.9.1`, which looks like it's tracking
   `vulnerability-scanner` rather than being deliberate. A bundle needs its own
   independent version. Recommend an explicit `rancher.bundles[].version` (as in 7.2)
   and leaving root `version` alone.

3. ~~**Does bundling force `prime-only` on `network`?**~~ **Moot if the suite is
   `vulnerability-scanner` + `runtime-enforcer` only** — both are already
   `prime-only: true` with identical compatibility ranges. Only reopens if `network`
   joins the bundle. Note also that `prime-only` turns out to be a display label with
   no install gate behind it (see 6.3), so the stakes are lower than assumed.

4. **Is all-or-nothing upgrade acceptable?** Every member re-deploys on any bundle
   upgrade. Fine for extensions (they're static assets, no workload restarts), but it
   means a patch to one extension bumps the version users see for all three.

5. **Migration for existing standalone installs.** Users with `vulnerability-scanner`
   0.9.1 installed will hit the Helm ownership error on bundle install. Options: document
   "uninstall first", or implement 5.6 so the UI guides them. Recommend 5.6.

6. **Does this land in `shell` (versioned, needs a release) or stay repo-local?**
   The chart-generation changes are in `@rancher/shell`, so bundling becomes available to
   every extension repo and is gated on a shell release. That's the right place for it,
   but it makes this a `rancher/dashboard` PR with the usual backport questions, not
   something `security-ui-exts` can land alone.
   **For the POC this does not bind** — see [§11](#10-poc-on-a-fork). The umbrella
   assembly can live as a repo-local script, so nothing needs publishing to npm.

7. **Should the bundle's annotations propagate onto members?** Answered in
   [§6.4](#64-should-the-bundles-annotations-propagate-down-to-the-members) — yes, onto
   each member's `values.plugin.metadata`, excluding `display-name` and `icon`; no, onto
   subchart `Chart.yaml`, where it has no effect.

8. **How does a *loaded* member learn it is part of a bundle?** It can't, today, without
   baking it into `pkg/*/package.json` at build time (Option B in
   [§4.1](#41-correction-this-is-not-a-single-channel)). The recommended cross-reference
   fix means the Dashboard knows, but the extension's own runtime code does not. Fine
   for everything proposed here; revisit if an extension ever needs to render
   suite-aware UI.

---

## 9. Verification plan

1. ~~**Chart-only smoke test, no Dashboard changes.**~~ **DONE** — see
   [§9.1](#91-smoke-test-results-run-2026-10-27).
2. **Install it on a real Rancher.** Confirm both members load at runtime (validating
   section 2 empirically), then confirm the Extensions screen misbehaves exactly as 3.1 /
   3.2 / 3.3 predict. Any divergence means the analysis is wrong somewhere.
3. **Then** build the Dashboard changes against that known-bad baseline.
4. Unit coverage for the new branches in `index.vue` and `UninstallExtensionDialog.vue` —
   existing suites at `pages/c/_cluster/uiplugins/__tests__/index.test.ts` and
   `dialog/__tests__/` are the obvious homes.

### 9.1 Smoke test results (run 2026-10-27)

Method: reproduced `publish`'s per-package chart generation by running the real
`helm/scripts/patch` and `helmpatch` from `@rancher/shell@3.0.13` against the actual
`pkg/vulnerability-scanner/package.json` and `pkg/runtime-enforcer/package.json`, then
vendored both into a hand-written umbrella and rendered it.

| # | Check | Result |
|---|---|---|
| 1 | Two `UIPlugin` CRs render from one umbrella | **Pass** |
| 2 | `metadata.name` == `spec.plugin.name` == package name | **Pass** — `vulnerability-scanner`, `runtime-enforcer` |
| 3 | `spec.plugin.version` is the *member's* version | **Pass** — `0.9.1` / `0.1.0`, off a bundle versioned `1.0.0` |
| 4 | …without needing `plugin.versionOverride` | **Pass** — contradicts the original §6.2 |
| 5 | Endpoints resolve per member, not per bundle | **Pass** |
| 6 | Bundle marker deep-merges into `plugin.metadata` | **Pass** — existing annotations preserved |
| 7 | Omitting `fullnameOverride` breaks it as §3.4 predicts | **Pass** — yields `security-ui-exts-vulnerability-scanner` |
| 8 | `condition: <pkg>.enabled` disables a member | **Pass** — 1 CR instead of 2 |
| 9 | `helm lint` | **Pass** (one benign "templates/ does not exist" warning — expected, the parent has no templates) |
| 10 | `helm package` with vendored subcharts, no network | **Pass** — archive is self-contained |
| 11 | `helm repo index` produces a usable entry | **Pass** |
| 12 | Only the *parent's* annotations reach `index.yaml` | **Confirmed** — subchart annotations are invisible to the catalog |
| 13 | `dependencies` (name + version + condition) reaches `index.yaml` | **Confirmed** — see §5.6 |

**Caveat:** run with **Helm v4.2.3** locally; Rancher installs via a Helm v3 library.
Every construct used here (vendored subcharts, `condition`, deep value merge,
`fullnameOverride`) is long-stable across v3 and v4, so the risk is low — but this has
not been re-run on v3.

**Not covered by this test:** anything requiring a cluster. Section 2's "the backend is
indifferent" claim, CR cache/`Ready` status, actual UMD loading, and the Extensions
screen failure modes all still need step 2.

---

## 10. POC on a fork

> This section was added after the feasibility check on 2026-10-27. It describes the
> **minimum** path to a working demonstration, deliberately skipping everything in §7.

**Verdict: feasible entirely with local changes on two forks. No `@rancher/shell`
release, no `rancher/dashboard` PR, no workflow changes.**

### 10.1 Scope

Two members only — `vulnerability-scanner` (0.9.1) and `runtime-enforcer` (0.1.0) —
with the member list as a configurable variable. `network` is excluded, which removes
open question 3 and makes the §6.3 range intersection trivial.

### 10.2 Why no shell release is needed

- `publish` **already** accepts a positional extension list, so the configurable member
  list exists today:
  ```
  yarn publish-pkgs -s <you>/security-ui-exts -b gh-pages vulnerability-scanner runtime-enforcer
  ```
- Umbrella assembly is **pure post-processing** on what `publish` leaves in `charts/`.
  It can be a repo-local script plus one `package.json` entry — nothing in
  `node_modules/@rancher/shell` is touched.
- The release-tag filter only engages when `-t` is passed. Omit it locally and there is
  no constraint on what gets built.
- All of §7 (the `-B`/`-N` flags, `parse-tag-name` bundle mode, `build-extension-charts.yml`
  inputs) is **productionisation**. Skip it; push to `gh-pages` by hand.

### 10.3 Minimum Dashboard change set

Run the dashboard fork with `yarn dev`, which serves `shell/` from source — UI changes
are live with no npm publish.

| Needed | Section | Why |
|---|---|---|
| **Yes** | 5.2 `wirePluginCRToChart` bundle branch | closes card source C (§3.7) — without it the bundle card never shows as installed and you get orphans |
| **Yes** | 5.3 suppress member cards — **use the rewritten version** | closes card source B (§3.7) — mandatory for the one-card requirement |
| **Yes** | 5.4 `UninstallExtensionDialog` release-name resolution | otherwise uninstall is a silent no-op |
| No | 5.5 `noAuth` hoisting | neither package sets `rancher.noAuth`; a root `plugin.noAuth: false` in the parent values is enough |
| No | 5.6 conflict guard | only matters if a member is already installed standalone — just don't, for the POC |
| No | 5.7 card presentation | cosmetic |

### 10.4 Prerequisites

1. **Create a `gh-pages` branch on the fork.** `publish` hard-exits if `-b gh-pages` is
   passed and the branch does not exist. An orphan branch is fine.
   GitHub **Pages itself is not required** — a `ClusterRepo` pointed at
   `https://raw.githubusercontent.com/<you>/security-ui-exts/gh-pages/` resolves
   `index.yaml` and the relative `assets/...` URLs directly, and that is already the URL
   shape `helmpatch` generates from `-s <you>/security-ui-exts`. The repo must be public.
2. **A real Rancher** to install against. Unavoidable for steps 2–3 of the verification
   plan — the backend index, CR cache status and actual JS loading cannot be faked.
3. **A compatible dashboard fork branch.** The extensions build against
   `@rancher/shell@3.0.13`; base the fork on a branch whose `shell/package.json` is
   close to that, or the UMD may fail to initialise against a mismatched shell API.

### 10.5 Order of work

1. Create `gh-pages` on the fork.
2. `yarn publish-pkgs -s <you>/security-ui-exts -b gh-pages vulnerability-scanner runtime-enforcer`
   and push the result. Confirm both extensions install and load **standalone** first —
   this proves the hosting before any bundling is involved.
3. Add the repo-local umbrella script; publish the bundle to the same `gh-pages`.
4. Install the bundle on Rancher with an **unmodified** Dashboard. Confirm §3.1 / 3.2 /
   3.3 reproduce exactly as described. If they don't, stop — the analysis is wrong.
5. Apply the three Dashboard changes on the fork and re-test against the same install.
6. **Acceptance check:** exactly one card on the Extensions screen, labelled
   `SUSE Security`. It installs both members, shows installed state with the bundle's
   version, and uninstalls both. Anything else fails the requirement in §3.7.

Note on step 2: it publishes the standalone member charts, which is deliberate — it
isolates "is the hosting working" from "is the bundling working". Clear `assets/` of the
member `.tgz` files and re-index before step 3 so the final state has the bundle entry
only.

---

## 11. Recommendation

Umbrella charts are **viable**, and the chart mechanics are now proven rather than
assumed (§9.1). The backend is genuinely indifferent — still the main outstanding risk,
since that part remains source-reading only. The work is confined to chart generation,
the build script, and roughly four call sites in the Dashboard. None of it is
architecturally hard.

The one genuine surprise from verification is §3.3 / §4.1: `spec.plugin.metadata` does
**not** reach the loaded-plugin list, so the "one marker, one channel" simplification
that motivated §4 only half-holds. The fix is still small — a four-line cross-reference
instead of a one-line read — but it is worth knowing before writing the PR, because the
original design read as though §5.3 were free.

Whether it's *worth* it depends on one question: do these extensions need independent
versioning and independent install?

- **If no** — merging the three packages into one `pkg/` directory registering three
  products gives you one chart, one CR, one install, with **zero** changes to the
  Dashboard or the build pipeline. Strictly less work and less risk than bundling, and it
  reaches the same user-facing outcome.
- **If yes** — bundling is the right call, and it's worth doing properly in `shell` so
  other extension repos get it too. Partner and community extension repos with multiple
  related extensions have the same problem.

The umbrella approach is also the only one of the two that survives a future where the
three extensions are released on different cadences — so if there's any doubt about
merging being permanent, bundling is the safer investment.
