# StackAdapt

A custom [Google Tag Manager (GTM)](https://tagmanager.google.com/) tag template (`.tpl`) that loads the **StackAdapt** self-serve advertising platform's conversion tracking pixel (`saq`) and fires a conversion event. It packages StackAdapt's native `saq` bootstrapping snippet into a reusable, configurable GTM template, so you no longer need to hand-maintain a Custom HTML tag with a hard-coded StackAdapt Pixel/Conversion ID.

## What is StackAdapt?

[StackAdapt](https://www.stackadapt.com/) is a self-serve programmatic advertising platform (native, display, video, connected TV, and audio ads). Advertisers place a JavaScript conversion pixel on their site to track conversions attributable to StackAdapt ad campaigns. The native snippet looks like this:

```html
<script type="text/javascript">
!function(b, e, f, g, a, c, d) {
  b.saq || (a = b.saq = function() {
    a.callMethod ? a.callMethod.apply(a, arguments) : a.queue.push(arguments)
  }, b._saq || (b._saq = a), a.push = a, a.loaded = !0, a.version = "1.0", a.queue = [],
  c = e.createElement(f), c.async = !0, c.src = g,
  d = e.getElementsByTagName(f)[0], d.parentNode.insertBefore(c, d))
}(window, document, "script", "https://tags.srv.stackadapt.com/events.js");
saq("conv", {{StackAdapt Pixel ID}});
</script>
```

This pattern — defining a `window.saq` stub/queue function, asynchronously loading `events.js`, and then calling `saq("conv", pixelId)` to register a conversion — is what this template reimplements using GTM's supported sandbox APIs.

## Why use a template instead of Custom HTML?

- **Configurable fields** — the StackAdapt Pixel/Conversion ID, the tracking script URL, and the `saq` version string are exposed as template fields with input validation, instead of being hard-coded values buried in an IIFE.
- **Sandboxed, permissioned execution** — the template runs in GTM's sandboxed JavaScript environment and only requests the exact permissions it needs (script injection restricted to StackAdapt's tag domain, and access to a small, explicit set of `saq`-related global properties).
- **Native queue behavior preserved** — the template recreates the `saq` stub function, `saq.push`, `saq.loaded`, `saq.version`, and `saq.queue` properties using `createQueue`/`setInWindow`, so any calls made to `saq(...)` before the real script finishes loading are queued exactly as they would be with the original snippet.
- **Reusability** — once imported, the template can be used to create any number of StackAdapt conversion tags (e.g., per campaign, page, or event) without duplicating raw HTML/JS.
- **Safer governance** — GTM shows reviewers exactly what a template is allowed to do (which URLs it can load scripts from, which global variables it can read/write) before it's published, which is harder to audit with an open Custom HTML tag.

## Repository contents

| File | Description |
|---|---|
| `StackAdapt.tpl` | The GTM community template gallery source file. Contains the template metadata, configurable fields, sandboxed JavaScript logic, required web permissions, and (empty) test scenarios. |

## Template fields

When you create a tag from this template in GTM, you'll configure three fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `stackAdaptId` | Text | Yes | Your StackAdapt Pixel/Conversion ID, passed as the second argument to `saq("ts", id)` when the tag fires. Must be non-empty. |
| `stackAdaptScriptUrl` | Text | Yes (has a default) | The URL of the StackAdapt tracking script. Defaults to `https://tags.srv.stackadapt.com/events.js`. Has a regex validator intended to restrict it to an `https://` StackAdapt-hosted `.js` file (see [Known quirk](#known-quirk-in-the-url-validator) below). |
| `stackAdaptVersion` | Text | Yes (has a default) | The version string exposed on `window.saq.version`. Defaults to `1`, mirroring the native snippet's `a.version = "1.0"`. |

## How it works

The template's sandboxed JavaScript (`___SANDBOXED_JS_FOR_WEB_TEMPLATE___`) does the following:

1. Reads the `stackAdaptId` and `stackAdaptScriptUrl` field values.
2. Creates a GTM-managed command queue named `saq` via `createQueue`.
3. Defines `window.saq` as a function that pushes its arguments onto that queue (`setInWindow('saq', ...)`), replicating the stub function from StackAdapt's native snippet.
4. Sets the companion properties expected by the real StackAdapt script once it loads:
   - `saq.push` — same queuing function as `saq` itself.
   - `saq.loaded` — set to `true`.
   - `saq.version` — set from the `stackAdaptVersion` field.
   - `saq.queue` — initialized as an empty array.
5. Defines an `initializeStackAdapt` helper that calls `saq('ts', stackAdaptId)` via `callInWindow` — this fires a **track site (`ts`)** style call registering the configured ID, once the script has confirmed loading.
6. Uses `injectScript` to asynchronously load the StackAdapt script from the configured URL, tagged with the `'stackAdaptScript'` queue name.
   - **On success**, it logs `StackAdapt script loaded successfully`, calls `initializeStackAdapt()` to fire the tracking call, then calls `data.gtmOnSuccess()`.
   - **On failure**, it logs `Failed to load StackAdapt script` and calls `data.gtmOnFailure()`, so GTM correctly reports the tag as failed.

This achieves the same end state as the native snippet — a fully initialized `saq` queue plus a registered tracking call — but built from GTM's supported sandbox primitives instead of a raw self-invoking function and inline DOM manipulation.

## Requested permissions

GTM template permissions are declared explicitly in `___WEB_PERMISSIONS___`. This template requests only what it needs:

- **`logging`** — allowed to write to the console, restricted to the `debug` environment only.
- **`access_globals`** — read/write/execute access limited to exactly five global identifiers:
  - `saq`
  - `saq.loaded`
  - `saq.push`
  - `saq.version`
  - `saq.queue`
- **`inject_script`** — allowed to inject `<script>` tags, restricted to `https://tags.srv.stackadapt.com/events.js`.

No other global variables, domains, or logging environments are accessible to the template.

## Getting started

### Import into Google Tag Manager

1. In your GTM container, go to **Templates** → **Tag Templates** → **New**.
2. Click the **⋮** (more actions) menu → **Import**.
3. Select the `StackAdapt.tpl` file from this repository.
4. Save the template.

### Create a tag from the template

1. Go to **Tags** → **New**.
2. Choose **StackAdapt** as the tag type (it will appear under your custom templates).
3. Enter your **StackAdapt Pixel/Conversion ID**.
4. Leave the script URL and version fields at their defaults unless StackAdapt has given you different values.
5. Set the appropriate trigger — typically a conversion event (e.g., **Purchase**, **Lead Submitted**) rather than All Pages, since this tag is meant to register conversions.
6. Save, preview, and test the tag before publishing.

### Verify it's working

- Use GTM's **Preview/Debug** mode and confirm the console shows `StackAdapt script loaded successfully`, with no `gtmOnFailure` calls.
- Open your browser's developer tools **Network** tab and confirm `events.js` loads with a `200` status from `tags.srv.stackadapt.com`.
- Check that `window.saq` is defined and `window.saq.loaded === true` in the console after the page loads.
- Confirm the conversion appears in your StackAdapt dashboard for your test session.

## Notes

- This template was created on 6/9/2025.
- The `___TESTS___` section currently contains no automated test scenarios (`scenarios: []`). Contributions adding test coverage via GTM's template testing framework are welcome.
- This is an unofficial, community-built template and is not published or endorsed by StackAdapt. Always review sandboxed template code and requested permissions before importing third-party templates into a production GTM container.

## License

No license file is currently included in this repository. Check with the repository owner before reusing this template in a commercial or redistributed context.
