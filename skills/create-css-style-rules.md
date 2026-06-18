# Programmatically Managing CSS Rules in Sitevision (Edit API)

Sitevision stores the site-wide CSS rule library (shown in any layer's **Properties → Style (CSS) → CSS rules** dropdown) in one server object: the **decorationsTab**. All operations = GET that object, modify its `cssClasses` array, PUT it back. Run from browser JS console **while logged into the Sitevision editor**.

## Step 1 — Get your IDs

`SITE_NODE_ID` comes from the editor URL `https://<host>/edit/<SITE_NODE_ID>/...`. `DECORATION_NODE_ID` = the node owning decorations; **get it reliably from a `nodeLock` request URL**, not from performance entries (those proved unreliable and gave `undefined`).

```js
window.SITE = location.pathname.split('/')[2];
window.BASE = "/edit-api/1/" + window.SITE;
// NODE: open Settings(gear)→Site settings once, then in DevTools Network find the
// request .../<NODE>/nodeLock  — copy that <NODE> id. Or grab via Resource Timing:
window.NODE = performance.getEntriesByType('resource')
  .map(e => (e.name.match(/\/([^/]+)\/nodeLock/) || [])[1])
  .filter(Boolean).pop();
window.URL_TAB = window.BASE + "/" + window.NODE + "/decorationsTab/" + window.NODE;
```

Sanity check: `await getState()` (Step 4) must return an object with `cssClasses`. If you get **500** with `.../undefined/decorationsTab/undefined`, your NODE didn't resolve — set it manually from the nodeLock URL.

## Step 2 — Auth

GET works with session cookie alone. All mutations (POST/PUT/DELETE) → **403** unless headers include:
`X-CSRF-Token` (32 chars), `X-Requested-With: XMLHttpRequest`, `Content-Type: application/json; charset=UTF-8`, `Accept: application/json`.

## Step 3 — Capture CSRF token

Not in any global/meta. Intercept from a GUI mutation. Paste first:

```js
window.__csrf = null;
const _o = XMLHttpRequest.prototype.open;
XMLHttpRequest.prototype.open = function(m,u,...r){ this.__u=u; return _o.call(this,m,u,...r); };
const _s = XMLHttpRequest.prototype.setRequestHeader;
XMLHttpRequest.prototype.setRequestHeader = function(n,v){
  if (this.__u && this.__u.includes('/edit-api/') && n==='X-CSRF-Token') window.__csrf = v;
  return _s.call(this,n,v);
};
```

Then open Settings(gear) → Site settings (fires a nodeLock POST carrying the token). `window.__csrf` now set. **Cancel** the dialog to release the GUI's lock (else it collides with your script's lock). Re-capture after any editor reload — token is session-scoped.

## Step 4 — Helpers

```js
window.H = () => ({'Content-Type':'application/json; charset=UTF-8','Accept':'application/json',
  'X-CSRF-Token':window.__csrf,'X-Requested-With':'XMLHttpRequest'});
window.lock     = () => fetch(window.BASE+"/"+window.NODE+"/nodeLock",{method:'POST',headers:window.H()});
window.unlock   = () => fetch(window.BASE+"/"+window.NODE+"/nodeLock/"+window.NODE,{method:'DELETE',headers:window.H()});
window.getState = async () => (await fetch(window.URL_TAB,{headers:window.H()})).json();
window.putState = (s) => fetch(window.URL_TAB,{method:'PUT',headers:window.H(),body:JSON.stringify(s)});
```

Every mutation = `lock → getState → modify → putState → unlock`. A good PUT returns status 200 and body `{success:true}`.

## Step 5 — Data shape

GET returns `{decorations, archivedDecorations, cssClasses, archivedCssClasses}`. Rules live in `cssClasses`. Each entry:

```json
{ "id":"54...", "name":"Hero wrapper", "cssClassName":"hero-wrapper",
  "customCss":"padding: 10em 0;", "useCustomCss":true, "roleType":"allRoles" }
```

- `id` — **omit for new rules** (server generates). For updates, **keep it** (see Step 8).
- `name` — dropdown label; any chars OK.
- `cssClassName` — actual HTML class; **validated** (Step 6).
- `customCss` — **declarations only, no selector wrapper** (`padding:1rem;`, not `.p-4{...}`). Media queries nest LESS-style: `@media (max-width:1000px){ padding:0.5rem; }`.
- `useCustomCss` — `true` when supplying customCss.
- `roleType` — `"allRoles"`.

## Step 6 — Class name validation (gotcha)

`cssClassName` **cannot contain `/` or `.`** → **422** `{"success":false,"context":{"cssClasses":{"key":"cssClassNameInvalid"...}}}`. Sanitize, keep readable display name:

```js
const sanitize = n => n.replace(/\//g,'-').replace(/\./g,'_'); // w-1/2→w-1-2, p-0.5→p-0_5
const makeRule = (label,css) => ({name:label, cssClassName:sanitize(label), customCss:css, useCustomCss:true, roleType:"allRoles"});
```

## Step 7 — CREATE (chunked, idempotent)

Don't PUT everything at once — chunk 15–20 per PUT (~40KB payload was fine). Re-fetch each loop, dedupe by `cssClassName`.

```js
async function uploadRules(rules, CHUNK=15){
  const log=[];
  for(let i=0;i<rules.length;i+=CHUNK){
    await window.lock();
    const st = await window.getState();
    const have = new Set(st.cssClasses.map(c=>(c.cssClassName||'').toLowerCase()));
    let added=0;
    for(const r of rules.slice(i,i+CHUNK))
      if(!have.has(r.cssClassName.toLowerCase())){ st.cssClasses.push(r); have.add(r.cssClassName.toLowerCase()); added++; }
    const res = await window.putState(st);
    let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
    log.push({i,added,status:res.status,ok,total:st.cssClasses.length});
    await window.unlock();
    if(res.status!==200||!ok) break; // stop on failure; body names the bad class
  }
  return log;
}
// const result = await uploadRules([ makeRule("p-4","padding:1rem;"), makeRule("flex","display:flex;") ]);
```

Healthy run: every chunk `status:200`, `ok:true`, rising `total`.

## Step 8 — UPDATE & DELETE (tested ✓)

Same cycle. **Update = find existing entry, mutate in place, KEEP its `id`.** If you drop the id (or push a new same-name object), the server creates a **duplicate** — confirmed live (server permits duplicate class names).

```js
// UPDATE — preserves id, no duplicate
async function updateRule(cssClassName, patch){
  await window.lock();
  const st = await window.getState();
  const rule = st.cssClasses.find(c => c.cssClassName === cssClassName); // matched entry keeps its id
  if (rule) Object.assign(rule, patch);                                  // edit in place
  const res = await window.putState(st);
  let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
  await window.unlock();
  return { found: !!rule, status: res.status, ok };
}
// updateRule("p-4", { customCss:"padding:1.25rem;", name:"p-4" });

// DELETE — hard removal (does NOT go to archivedCssClasses; verified)
async function deleteRule(cssClassName){
  await window.lock();
  const st = await window.getState();
  const before = st.cssClasses.length;
  st.cssClasses = st.cssClasses.filter(c => c.cssClassName !== cssClassName); // removes ALL matches
  const res = await window.putState(st);
  let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
  await window.unlock();
  return { removed: before - st.cssClasses.length, status: res.status, ok };
}
```

Notes: renaming `cssClassName` obeys Step 6 validation, and **breaks any layer/page already using the old class** (only the definition changes, not references). `deleteRule` removes every entry with that class name — handy for clearing accidental duplicates.

## Step 9 — Verify

```js
const after = await window.getState();
const names = new Set(after.cssClasses.map(c=>c.cssClassName));
console.log({ total: after.cssClasses.length });
// for creates: const missing = expected.filter(n=>!names.has(n));  // should be []
```

Then reload the editor (in-memory model is stale until reload) → right-click any layer → Properties → Style (CSS) → type in the CSS rules field → changes appear.

## Gotchas (quick ref)

403 on mutation = missing `X-CSRF-Token`. 422 "count limit" = actually `/` or `.` in `cssClassName`. 500 with `undefined` in URL = NODE id didn't resolve (get it from a `nodeLock` URL). New rules omit `id`; **updates must keep `id`** or you get a duplicate (server allows dup class names). `customCss` = declarations only, no wrapper. Delete is a hard removal, not archived. Re-fetch + dedupe inside loops = re-runnable. Re-capture token after editor reload. Always Cancel the GUI Site-settings dialog before scripting so its lock doesn't collide with yours.