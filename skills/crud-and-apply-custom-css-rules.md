# CRUD + Apply Custom CSS Rules in Sitevision (Edit API)

Two server objects, run from the browser console **while logged into the editor**:

- **`decorationsTab`** holds the site-wide CSS rule **library** (the `cssClasses` array). Create/update/delete rule definitions here.
- **`layoutSettings/<LAYER>`** holds which rules a layer **applies** (the `cssClassNames` string of rule **node IDs**).

Workflow: define rules in the library → apply them to a layer.

## Step 1 — IDs & auth

```js
window.SITE = location.pathname.split('/')[2];      // from /edit/<SITE>/...
window.BASE = "/edit-api/1/" + window.SITE;
// NODE / DECO: open Settings(gear)→Site settings, find a `.../<NODE>/nodeLock`
// request in DevTools Network, copy that <NODE>. (Performance-entry tricks are unreliable.)
window.NODE = performance.getEntriesByType('resource')
  .map(e => (e.name.match(/\/([^/]+)\/nodeLock/) || [])[1]).filter(Boolean).pop();
window.URL_TAB = `${BASE}/${NODE}/decorationsTab/${NODE}`;
```

GET works with the session cookie alone. **All mutations need** these headers or you get **403**:
`X-CSRF-Token` (32 chars), `X-Requested-With: XMLHttpRequest`, `Content-Type: application/json; charset=UTF-8`, `Accept: application/json`.

CSRF isn't in any global/meta — intercept it from a GUI mutation. Paste first, then open Settings(gear)→Site settings (fires a nodeLock POST), then **Cancel** the dialog so its lock doesn't collide with yours:

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

## Step 2 — Helpers

```js
window.H = () => ({'Content-Type':'application/json; charset=UTF-8','Accept':'application/json',
  'X-CSRF-Token':window.__csrf,'X-Requested-With':'XMLHttpRequest'});
window.lock     = () => fetch(`${BASE}/${NODE}/nodeLock`,{method:'POST',headers:H()});
window.unlock   = () => fetch(`${BASE}/${NODE}/nodeLock/${NODE}`,{method:'DELETE',headers:H()});
window.getState = async () => (await fetch(URL_TAB,{headers:H()})).json();
window.putState = (s) => fetch(URL_TAB,{method:'PUT',headers:H(),body:JSON.stringify(s)});
```

Every library mutation = `lock → getState → modify → putState → unlock`. Good PUT = `200` + `{success:true}`.

## Step 3 — Rule shape & CRUD (library)

Each `cssClasses` entry: `{ id, name, cssClassName, customCss, useCustomCss, roleType }`.
- `id` — **omit when creating** (server generates); **keep when updating** or you get a duplicate (dup names allowed).
- `cssClassName` — real HTML class; **no `/` or `.`** → else **422** `cssClassNameInvalid`. `customCss` = declarations only, no selector wrapper (media queries nest LESS-style). `roleType` = `"allRoles"`.

```js
const sanitize = n => n.replace(/\//g,'-').replace(/\./g,'_');   // w-1/2→w-1-2, p-0.5→p-0_5
const makeRule = (label,css) => ({name:label, cssClassName:sanitize(label), customCss:css, useCustomCss:true, roleType:"allRoles"});

async function uploadRules(rules, CHUNK=15){                      // chunk ~15-20/PUT, idempotent
  const log=[];
  for(let i=0;i<rules.length;i+=CHUNK){
    await lock(); const st = await getState();
    const have = new Set(st.cssClasses.map(c=>(c.cssClassName||'').toLowerCase())); let added=0;
    for(const r of rules.slice(i,i+CHUNK))
      if(!have.has(r.cssClassName.toLowerCase())){ st.cssClasses.push(r); have.add(r.cssClassName.toLowerCase()); added++; }
    const res = await putState(st); let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
    log.push({i,added,status:res.status,ok,total:st.cssClasses.length}); await unlock();
    if(res.status!==200||!ok) break;                             // body names the bad class
  }
  return log;
}

async function updateRule(cssClassName, patch){                   // edit in place, keeps id → no dup
  await lock(); const st = await getState();
  const rule = st.cssClasses.find(c => c.cssClassName === cssClassName); if (rule) Object.assign(rule, patch);
  const res = await putState(st); let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
  await unlock(); return { found: !!rule, status: res.status, ok };
}

async function deleteRule(cssClassName){                          // hard remove (NOT archived), all matches
  await lock(); const st = await getState(); const before = st.cssClasses.length;
  st.cssClasses = st.cssClasses.filter(c => c.cssClassName !== cssClassName);
  const res = await putState(st); let ok=false; try{ ok=(await res.clone().json()).success; }catch(e){}
  await unlock(); return { removed: before - st.cssClasses.length, status: res.status, ok };
}
// uploadRules([ makeRule("p-4","padding:1rem;"), makeRule("flex","display:flex;") ])
// updateRule("p-4",{customCss:"padding:1.25rem;"});  deleteRule("p-4")
```

Renaming `cssClassName` only changes the definition — it **breaks layers/pages already referencing the old class**.

## Step 4 — Apply / remove rules on a layer

A layer's applied rules live in `layoutSettings.cssClassNames` — a comma-separated string of rule **node IDs**, not class names. Map names→IDs via `siteCssClassNameSelection` (its `value` = the rule node id). `LAYER` = layer node id from its Properties URL `/edit/<SITE>/<LAYER>/properties` (layer selection is client-side only — no API). `DECO` = decoration node id.

```js
const sel = await (await fetch(`${BASE}/${DECO}/siteCssClassNameSelection`,{headers:H()})).json();
const idOf = cn => (sel.find(x=>x.className===cn||x.name===cn)||{}).value;

async function setLayerClasses(LAYER, {add=[], remove=[]}={}) {
  const s = await (await fetch(`${BASE}/${SITE}/layoutSettings/${LAYER}`,{headers:H()})).json();
  let ids = (s.cssClassNames||'').split(',').filter(Boolean);
  remove.map(idOf).forEach(id => ids = ids.filter(x=>x!==id));
  add.map(idOf).forEach(id => { if(id && !ids.includes(id)) ids.push(id); });   // dedupe
  s.cssClassNames = ids.join(',');                                              // PUT whole object back
  const r = await fetch(`${BASE}/${SITE}/layoutSettings/${LAYER}`,{method:'PUT',headers:H(),body:JSON.stringify(s)});
  return {status:r.status, ...(await r.json())};
}
// setLayerClasses(LAYER,{add:['p-4','gap-6']});  setLayerClasses(LAYER,{remove:['grid-cols-3']})
```

PUT the **entire** layoutSettings object (path uses `<SITE>/<SITE>/layoutSettings/<LAYER>` — site id twice), or other layer settings are lost.

## Gotchas (quick ref)

- **403** = missing `X-CSRF-Token` (re-capture after any editor reload). **422 "count limit"** = actually `/` or `.` in `cssClassName`. **500 with `undefined` in URL** = NODE didn't resolve — set it from a nodeLock URL.
- New rules omit `id`; updates keep `id`. `customCss` = declarations only. Delete is permanent, not archived. Re-fetch + dedupe inside loops = re-runnable.
- `cssClassNames` on a layer stores node IDs — always map via `siteCssClassNameSelection`.
- Changes save to the working copy. Reload the editor (in-memory model is stale until reload) to see them; **publish** separately via right-click → Update.
