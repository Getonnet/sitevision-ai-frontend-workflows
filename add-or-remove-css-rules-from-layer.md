Done. I reverse-engineered the layer-level apply/remove API, tested it live (apply, remove, and restore all returned `{"success":true}`), and left Section 2 back in its original state. Here's the updated **fully programmatic** guide.

## Guide: Apply / Remove CSS Rules on a Layer (API, no GUI)

**How it works:** A layer's applied CSS rules are stored in its `layoutSettings` object under `cssClassNames` — a comma-separated string of CSS-rule **node IDs** (not class names). To apply/remove, you GET that object, edit the `cssClassNames` string, and PUT it back. Class names are mapped to node IDs via `siteCssClassNameSelection`.

**Prerequisites (capture once per session):**
- `SITE` = site node id (e.g. from editor URL `/edit/<SITE>`)
- `DECO` = decoration node id (the one used for the site's CSS rules)
- `LAYER` = target layer's node id (appears in the URL when its Properties dialog is open: `/edit/<SITE>/<LAYER>/properties`)
- `CSRF` token — read from `<meta name="csrf-token">` or capture from any editor XHR header `X-CSRF-Token`. Re-capture after any editor reload.

All mutations need headers: `X-CSRF-Token`, `X-Requested-With: XMLHttpRequest`, `Content-Type: application/json; charset=UTF-8`, `Accept: application/json`. Missing CSRF = 403.

**Steps:**

1. **Build name→ID map.** GET `/edit-api/1/<SITE>/<DECO>/siteCssClassNameSelection`. Returns array of `{name, className, value, assignable}` where `value` is the rule node id. Look up by `className` (or `name`).

2. **GET current layer settings.** GET `/edit-api/1/<SITE>/<SITE>/layoutSettings/<LAYER>`. Keep the **whole object**; you'll PUT it back unchanged except `cssClassNames`.

3. **Edit the list.** Split `cssClassNames` on `,`. To **apply**: push the rule's `value` id (dedupe — skip if present). To **remove**: filter the id out. Re-join with `,`.

4. **PUT it back.** PUT `/edit-api/1/<SITE>/<SITE>/layoutSettings/<LAYER>` with the full object (modified `cssClassNames`). Success = `200` + `{"success":true}`.

5. **Verify (optional).** GET layoutSettings again, confirm `cssClassNames` matches expectation.

**Reference snippet:**
```js
const base = `/edit-api/1/${SITE}`;
const H = {'Accept':'application/json','X-Requested-With':'XMLHttpRequest'};
const HW = {...H,'Content-Type':'application/json; charset=UTF-8','X-CSRF-Token':CSRF};

const sel = await (await fetch(`${base}/${DECO}/siteCssClassNameSelection`,{headers:H})).json();
const idOf = cn => (sel.find(x=>x.className===cn||x.name===cn)||{}).value;

async function setLayerClasses(LAYER, {add=[], remove=[]}={}) {
  const s = await (await fetch(`${base}/${SITE}/layoutSettings/${LAYER}`,{headers:H})).json();
  let ids = (s.cssClassNames||'').split(',').filter(Boolean);
  remove.map(idOf).forEach(id => ids = ids.filter(x=>x!==id));
  add.map(idOf).forEach(id => { if(id && !ids.includes(id)) ids.push(id); });
  s.cssClassNames = ids.join(',');
  const r = await fetch(`${base}/${SITE}/layoutSettings/${LAYER}`,{method:'PUT',headers:HW,body:JSON.stringify(s)});
  return {status:r.status, ...(await r.json())};
}
// apply:  setLayerClasses(LAYER,{add:['p-4','gap-6']})
// remove: setLayerClasses(LAYER,{remove:['grid-cols-3']})
```

**Notes / gotchas:**
- `cssClassNames` stores **node IDs**, not class names — always map through `siteCssClassNameSelection`.
- PUT the **entire** layoutSettings object, not just the changed field, or other layer settings can be lost.
- Re-fetch settings inside the loop and dedupe → idempotent / re-runnable.
- Use endpoint form `/<SITE>/<SITE>/layoutSettings/<LAYER>` (site id appears twice; layer id is the path segment).
- Getting `LAYER` programmatically: layer selection is client-side only (no API), so grab the id from the Properties URL or from a `nodeLock`/context-menu request URL.
- Publishing is separate — these changes save to the working copy; publish via the page's right-click → Update when ready.

This replaces the GUI-based apply/remove guide entirely. Want me to also fold this into a single combined "complete" guide alongside the create/update/delete one?