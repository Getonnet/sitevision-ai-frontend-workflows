# Programmatically Creating CSS Rules in Sitevision (Edit API)

Sitevision stores the site-wide CSS rule library (shown in any layer's **Properties → Style (CSS) → CSS rules** dropdown) in one server object: the **decorationsTab**. Fast method = GET that object, append to its `cssClasses` array, PUT it back. Run everything from the browser JS console **while logged into the Sitevision editor**.

## Step 1 — Get your IDs (no hardcoding)

Read them from the current editor URL so the script works on any site. Editor URL pattern: `https://<host>/edit/<SITE_NODE_ID>/...`

```js
const SITE = location.pathname.split('/')[2];   // SITE_NODE_ID from URL
const BASE = "/edit-api/1/" + SITE;
```

You also need the `DECORATION_NODE_ID` (node owning decorations/CSS rules). Find it: open Settings (gear) → Site settings → Decorations, watch Network — the `.../decorationsTab/<DECORATION_NODE_ID>` request reveals it. Capture programmatically:

```js
// after opening Site settings → Decorations once, the request fires; grab from performance entries
const NODE = performance.getEntriesByType('resource')
  .map(e => (e.name.match(/\/decorationsTab\/([^/?]+)/) || [])[1])
  .filter(Boolean).pop();
const URL_TAB = BASE + "/" + NODE + "/decorationsTab/" + NODE;
```

## Step 2 — Auth

GET works with session cookie alone. All mutations (POST/PUT/DELETE) → **403** unless you send:
- `X-CSRF-Token` (32 chars)
- `X-Requested-With: XMLHttpRequest`
- plus `Content-Type: application/json; charset=UTF-8`, `Accept: application/json`

## Step 3 — Capture CSRF token

Token not in any global/meta. Intercept the header from a GUI mutation. Paste first:

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

Then trigger a mutation: open Settings (gear) → Site settings (acquires a node lock = POST carrying token). `window.__csrf` now set. **Cancel** the dialog to release the GUI lock (else collides with your script's lock). Token is session-scoped — re-capture after any editor reload.

## Step 4 — Helpers

```js
const H = () => ({
  'Content-Type':'application/json; charset=UTF-8',
  'Accept':'application/json',
  'X-CSRF-Token':window.__csrf,
  'X-Requested-With':'XMLHttpRequest'
});
const lock   = () => fetch(BASE+"/"+NODE+"/nodeLock", {method:'POST', headers:H()});
const unlock = () => fetch(BASE+"/"+NODE+"/nodeLock/"+NODE, {method:'DELETE', headers:H()});
const getState = async () => (await fetch(URL_TAB,{headers:H()})).json();
const putState = (s) => fetch(URL_TAB,{method:'PUT', headers:H(), body:JSON.stringify(s)});
```

## Step 5 — Data shape

GET returns `{decorations, archivedDecorations, cssClasses, archivedCssClasses}`. Rules live in `cssClasses`. Each entry:

```json
{ "id":"54...", "name":"Hero wrapper", "cssClassName":"hero-wrapper",
  "customCss":"padding: 10em 0;", "useCustomCss":true, "roleType":"allRoles" }
```

Field rules:
- `id` — **omit for new rules** (server generates). Existing rules echo their id back unchanged.
- `name` — dropdown label, any chars OK.
- `cssClassName` — actual HTML class, **validated** (see Step 6).
- `customCss` — **declarations only, no selector wrapper** (write `padding:1rem;`, not `.p-4{...}`). Media queries nest LESS-style: `@media (max-width:1000px){ padding:0.5rem; }`.
- `useCustomCss` — `true` when supplying customCss.
- `roleType` — `"allRoles"`.

## Step 6 — Class name validation (big gotcha)

`cssClassName` **cannot contain `/` or `.`** → returns **422** `{"success":false,"context":{"cssClasses":{"key":"cssClassNameInvalid"...}}}`. Looks like a "rule count cap" but isn't. Sanitize while keeping readable display name:

```js
const sanitize = n => n.replace(/\//g,'-').replace(/\./g,'_'); // w-1/2→w-1-2, p-0.5→p-0_5
const makeRule = (label, css) => ({ name:label, cssClassName:sanitize(label), customCss:css, useCustomCss:true, roleType:"allRoles" });
```

## Step 7 — Build rules

```js
const newRules = [
  makeRule("p-4","padding:1rem;"),
  makeRule("flex","display:flex;"),
  makeRule("grid-cols-12","grid-template-columns:repeat(12,minmax(0,1fr));"),
];
```

## Step 8 — Upload loop (chunked, idempotent)

Don't PUT everything at once — chunk 15–20 per PUT. Re-fetch each loop, dedupe by `cssClassName` so re-runs are safe.

```js
async function uploadRules(rules, CHUNK=15){
  const log=[];
  for(let i=0;i<rules.length;i+=CHUNK){
    await lock();
    const st = await getState();
    const have = new Set(st.cssClasses.map(c=>(c.cssClassName||'').toLowerCase()));
    let added=0;
    for(const r of rules.slice(i,i+CHUNK)){
      if(!have.has(r.cssClassName.toLowerCase())){ st.cssClasses.push(r); have.add(r.cssClassName.toLowerCase()); added++; }
    }
    const res = await putState(st);
    let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
    log.push({i,added,status:res.status,ok,total:st.cssClasses.length});
    await unlock();
    if(res.status!==200||!ok) break; // stop on failure; inspect body for offending class name
  }
  return log;
}
const result = await uploadRules(newRules); console.log(result);
```

Healthy run: every chunk `status:200`, `ok:true`, rising `total`. A 422 stops the loop — the response names the bad `cssClassName`.

## Step 9 — Verify

```js
const after = await getState();
const names = new Set(after.cssClasses.map(c=>c.cssClassName));
const missing = newRules.map(r=>r.cssClassName).filter(n=>!names.has(n));
console.log({total:after.cssClasses.length, missing}); // missing should be []
```

Then reload editor (in-memory model is stale until reload) → right-click any layer → Properties → Style (CSS) → type in CSS rules field → new rules appear.

## Gotchas (quick ref)

403 on mutation = missing `X-CSRF-Token`, not permissions. 422 "count limit" = actually `/` or `.` in `cssClassName`. New rules omit `id`. `customCss` = declarations only, no wrapper. Re-fetch + dedupe inside loop = re-runnable. Re-capture token after editor reload.