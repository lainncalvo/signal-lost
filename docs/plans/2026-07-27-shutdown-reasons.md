# Shutdown Reasons Implementation Plan

> **For agentic workers:** Steps use checkbox (`- [ ]`) syntax for tracking. Implement task-by-task, verifying after each.

**Goal:** Add a researched shutdown reason — a tag, a one-line explanation, and key figures — to all 109 entries, and surface it through a column, expandable rows, a filter, and two new charts.

**Architecture:** Everything lives in the single self-contained `index.html`. Three optional fields (`why`, `note`, `figs`) are added to each object in the `DATA` array. All stats, charts and filters continue to be derived from `DATA` at runtime, so adding the fields automatically feeds the new views. No build step, no dependencies.

**Tech Stack:** Vanilla HTML/CSS/JS. Verification via browser DOM assertions (this project has no test runner; correctness is asserted by querying the rendered DOM, which is the established pattern here).

**Spec:** `docs/specs/2026-07-27-shutdown-reasons-design.md`

---

## File Structure

| File | Responsibility |
|---|---|
| `index.html` | The entire site. All changes land here. |
| `docs/specs/2026-07-27-shutdown-reasons-design.md` | The approved design. Read-only reference. |
| `README.md` | Contributor docs — must document the new fields. |

`index.html` is a single file by design (it must work as one static asset). Within it, changes are confined to five regions: the `DATA` array, the constants block near `CLOSURE_LABEL`, the CSS block, the registry markup, and the render/filter functions.

**Verification helper.** Every verification step below runs JS in the browser against `http://localhost:8123/index.html`. Start the server first if it is not running:

```bash
cd "/Users/laincalvo/Documents/Side Projects/Crypto Project Shut Down 2026 - Present" && python3 -m http.server 8123
```

---

### Task 1: Merge researched reasons into DATA

**Files:**
- Modify: `index.html` (the `DATA` array)

- [ ] **Step 1: Consolidate the research results**

The nine research batches land in `scratchpad/whyres/batch{1..9}.json`, each an array of `{n, why, note, figs, confidence}`. Consolidate and check coverage:

```bash
cd "/Users/laincalvo/Documents/Side Projects/Crypto Project Shut Down 2026 - Present"
export SP="/private/tmp/claude-501/-Users-laincalvo-Documents-Side-Projects-Crypto-Project-Shut-Down-2026---Present/0932e6fb-403d-463d-abc4-82422d6e151b/scratchpad"
node -e '
const fs=require("fs");const SP=process.env.SP;
let all=[];for(let i=1;i<=9;i++)all=all.concat(JSON.parse(fs.readFileSync(SP+"/whyres/batch"+i+".json","utf8")));
const h=fs.readFileSync("index.html","utf8");
const names=[...h.matchAll(/\{n:"([^"]+)"/g)].map(m=>m[1]);
const got=new Set(all.map(r=>r.n));
console.log("researched:",all.length,"of",names.length);
console.log("missing:",names.filter(n=>!got.has(n)).join(", ")||"none");
const tally={};all.forEach(r=>tally[r.why]=(tally[r.why]||0)+1);
console.log("by reason:",JSON.stringify(tally,null,1));
fs.writeFileSync(SP+"/why_merged.json",JSON.stringify(all,null,1));
'
```

Expected: `researched: 109 of 109`, `missing: none`. If any are missing, re-run research for those before continuing.

- [ ] **Step 2: Validate every tag is in the taxonomy**

```bash
node -e '
const fs=require("fs");
const ok=new Set(["funding","exploit","traction","market","regulatory","insolvency","acquired","dependency","voluntary","unknown"]);
const all=JSON.parse(fs.readFileSync(process.env.SP+"/why_merged.json","utf8"));
const bad=all.filter(r=>!ok.has(r.why));
console.log("invalid tags:",bad.length?bad.map(r=>r.n+"="+r.why).join(", "):"none");
const longNotes=all.filter(r=>r.note&&r.note.length>130);
console.log("notes over 130 chars:",longNotes.length?longNotes.map(r=>r.n).join(", "):"none");
const unknownWithNote=all.filter(r=>r.why==="unknown"&&r.note);
console.log("unknown but has note (should be 0):",unknownWithNote.length);
'
```

Expected: `invalid tags: none`. Fix any invalid tag by mapping it to the closest valid one or to `unknown`. Trim any note over 130 chars. Drop notes on `unknown` entries.

- [ ] **Step 3: Write the fields into DATA**

```bash
node -e '
const fs=require("fs");
const all=JSON.parse(fs.readFileSync(process.env.SP+"/why_merged.json","utf8"));
const by={};all.forEach(r=>by[r.n]=r);
let h=fs.readFileSync("index.html","utf8");
let n=0;
h=h.replace(/\{n:"([^"]+)",(\s*)iso:"([^"]+)",(\s*)day:(true|false),?(\s*)url:(null|"[^"]*"),(\s*)c:"([^"]+)",(\s*)cat:"([^"]+)"\}/g,
 function(m,name,s1,iso,s2,day,s3,url,s4,c,s5,cat){
   const r=by[name]; if(!r) return m;
   n++;
   let extra=", why:"+JSON.stringify(r.why);
   if(r.note && r.why!=="unknown") extra+=", note:"+JSON.stringify(r.note);
   if(r.figs && r.figs.length) extra+=", figs:"+JSON.stringify(r.figs);
   return "{n:\""+name+"\","+s1+"iso:\""+iso+"\","+s2+"day:"+day+","+s3+"url:"+url+","+s4+"c:\""+c+"\","+s5+"cat:\""+cat+"\""+extra+"}";
 });
fs.writeFileSync("index.html",h);
console.log("rows updated:",n);
'
```

Expected: `rows updated: 109`.

- [ ] **Step 4: Verify the page still parses and renders**

```bash
node -e 'const h=require("fs").readFileSync("index.html","utf8");console.log("entries:",(h.match(/\{n:"/g)||[]).length,"| with why:",(h.match(/why:"/g)||[]).length);'
```

Expected: `entries: 109 | with why: 109`.

Then in the browser at `http://localhost:8123/index.html`, run:

```js
JSON.stringify({rows: document.querySelectorAll('#rows .row').length, errors: 0})
```

Expected: `rows: 109`. Also confirm the console is clean (no errors).

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "data: add researched shutdown reason, note and figures to all 109 entries"
```

---

### Task 2: Reason constants, labels and colors

**Files:**
- Modify: `index.html` (constants block, next to `CLOSURE_LABEL`; CSS block)

- [ ] **Step 1: Add the label and order constants**

Immediately after the existing `CLOSURE_ORDER` line, add:

```js
  var WHY_LABEL = {
    funding:    "Ran out of funding",
    exploit:    "Exploit / hack",
    traction:   "No traction",
    market:     "Market contraction",
    regulatory: "Regulatory / sanctions",
    insolvency: "Insolvency / bankruptcy",
    acquired:   "Acquired / absorbed",
    dependency: "Dependency shut down",
    voluntary:  "Wound down by choice",
    unknown:    "Not reported"
  };
  var WHY_SHORT = {
    funding:"Funding", exploit:"Exploit", traction:"No traction", market:"Market",
    regulatory:"Regulatory", insolvency:"Insolvency", acquired:"Acquired",
    dependency:"Dependency", voluntary:"By choice", unknown:"Not reported"
  };
  var WHY_ORDER = ["funding","traction","exploit","market","voluntary","acquired","dependency","insolvency","regulatory","unknown"];
```

- [ ] **Step 2: Add the chip CSS**

Add to the CSS block, after the `.cat-tag` rule:

```css
  .why-chip { font-family: var(--font-mono); font-size: 11px; padding: 3px 8px;
    border: 1px solid currentColor; white-space: nowrap; display: inline-block;
    letter-spacing: 0.02em; opacity: 0.95; }
  .why-funding    { color: #d98c2b; }
  .why-exploit    { color: var(--dead); }
  .why-traction   { color: #8c93a1; }
  .why-market     { color: #6f9bd1; }
  .why-regulatory { color: #b06fd1; }
  .why-insolvency { color: #d1566f; }
  .why-acquired   { color: #4fae9b; }
  .why-dependency { color: #c9a227; }
  .why-voluntary  { color: #7f9c6d; }
  .why-unknown    { color: var(--faint); border-style: dashed; }
```

- [ ] **Step 3: Verify constants load**

In the browser console:

```js
typeof WHY_LABEL
```

This returns `undefined` because the script is wrapped in an IIFE — that is expected and correct. Instead verify no syntax error was introduced:

```js
JSON.stringify({rows: document.querySelectorAll('#rows .row').length})
```

Expected: `rows: 109`. If it returns 0 or the page is blank, a syntax error was introduced — check the console.

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "feat: add reason taxonomy labels and chip styles"
```

---

### Task 3: Add the Why column to the registry table

**Files:**
- Modify: `index.html` (CSS grid, `.rhead` markup, `rowHTML`)

- [ ] **Step 1: Widen the grid to six columns**

Replace:

```css
  .rhead, .row { display: grid; grid-template-columns: 58px 1.7fr 1fr 118px 210px; align-items: center; }
```

with:

```css
  .rhead, .row { display: grid; grid-template-columns: 52px 1.45fr 0.95fr 1fr 112px 210px; align-items: center; }
```

The announcement column stays at 210px — it was widened previously to stop the button overflowing, and must not shrink.

- [ ] **Step 2: Add the header cell**

Replace:

```html
          <div>#</div><div>Project</div><div>Category</div><div>Date</div><div>Announcement</div>
```

with:

```html
          <div>#</div><div>Project</div><div>Category</div><div>Why</div><div>Date</div><div>Announcement</div>
```

- [ ] **Step 3: Render the chip in `rowHTML`**

In `rowHTML`, after the `c-cat` div and before the `c-date` div, insert:

```js
      '<div class="c-why"><span class="why-chip why-' + esc(d.why || "unknown") + '">' +
        esc(WHY_SHORT[d.why] || WHY_SHORT.unknown) + '</span></div>' +
```

- [ ] **Step 4: Verify the column renders and nothing overflows**

In the browser:

```js
(function(){
  var chips=document.querySelectorAll('#rows .why-chip');
  var btn=document.querySelector('#rows .btn-link');
  var cell=btn.closest('.c-link');
  return JSON.stringify({
    chips: chips.length,
    sample: chips[0].textContent,
    buttonOverflowsCell: Math.round(btn.getBoundingClientRect().right - cell.getBoundingClientRect().right),
    pageOverflow: document.documentElement.scrollWidth - document.documentElement.clientWidth
  });
})()
```

Expected: `chips: 109`, `buttonOverflowsCell` ≤ 0, `pageOverflow: 0`. If `buttonOverflowsCell` is positive, apply the spec's fallback: drop the `Why` column and render the chip beneath the project name inside the `Project` cell instead.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "feat: add Why column to the registry table"
```

---

### Task 4: Expandable rows showing the note and figures

**Files:**
- Modify: `index.html` (CSS, `rowHTML`, event wiring)

- [ ] **Step 1: Add the expansion CSS**

```css
  .row.has-detail { cursor: pointer; }
  .row.has-detail:hover .c-name { color: var(--accent-ink); }
  .detail { grid-column: 1 / -1; border-top: 1px dashed var(--border-2);
    background: var(--panel-2); padding: 16px; display: none; }
  .row.open .detail { display: block; }
  .row.open { background: var(--panel-2); }
  .detail p { margin: 0 0 10px; font-size: 14px; color: var(--text); max-width: 78ch; }
  .detail .figs { display: flex; flex-wrap: wrap; gap: 10px; }
  .detail .fig { font-family: var(--font-mono); font-size: 12px; border: 1px solid var(--border-2);
    padding: 5px 10px; color: var(--dim); }
  .detail .fig b { color: var(--accent-ink); font-weight: 700; margin-left: 6px; }
  .detail .why-full { font-family: var(--font-mono); font-size: 11px; letter-spacing: 0.08em;
    text-transform: uppercase; color: var(--faint); margin: 0 0 8px; }
```

- [ ] **Step 2: Render the detail panel and mark expandable rows**

In `rowHTML`, change the opening div to add the class and accessibility attributes when a note exists:

```js
    var hasDetail = !!d.note;
    var open = '<div class="row' + (hasDetail ? ' has-detail' : '') + '" role="row"' +
      (hasDetail ? ' tabindex="0" aria-expanded="false"' : '') +
      ' style="animation-delay:' + Math.min(i*22,500) + 'ms">';
```

and append the detail block just before the closing `</div>` of the row:

```js
    var detail = hasDetail
      ? '<div class="detail"><p class="why-full">' + esc(WHY_LABEL[d.why] || "") + '</p><p>' + esc(d.note) + '</p>' +
        (d.figs && d.figs.length
          ? '<div class="figs">' + d.figs.map(function (f) {
              return '<span class="fig">' + esc(f.k) + '<b>' + esc(f.v) + '</b></span>';
            }).join("") + '</div>'
          : "") +
        '</div>'
      : "";
```

Return `open + …cells… + detail + '</div>'`.

- [ ] **Step 3: Wire click and keyboard toggling**

Add near the other event listeners:

```js
  function toggleRow(row) {
    if (!row || !row.classList.contains("has-detail")) return;
    var isOpen = row.classList.contains("open");
    Array.prototype.forEach.call(document.querySelectorAll("#rows .row.open"), function (r) {
      r.classList.remove("open"); r.setAttribute("aria-expanded","false");
    });
    if (!isOpen) { row.classList.add("open"); row.setAttribute("aria-expanded","true"); }
  }
  el("rows").addEventListener("click", function (e) {
    if (e.target.closest("a")) return;          // let announcement links work
    toggleRow(e.target.closest(".row"));
  });
  el("rows").addEventListener("keydown", function (e) {
    if (e.key !== "Enter" && e.key !== " ") return;
    var row = e.target.closest(".row.has-detail");
    if (!row) return;
    e.preventDefault();
    toggleRow(row);
  });
```

Only one row is open at a time. Because `apply()` re-renders `#rows` wholesale, expansion state resets on any filter change automatically — no extra code needed.

- [ ] **Step 4: Verify expansion works**

In the browser:

```js
(function(){
  var r=document.querySelector('#rows .row.has-detail');
  var before=getComputedStyle(r.querySelector('.detail')).display;
  r.click();
  var after=getComputedStyle(r.querySelector('.detail')).display;
  var openCount=document.querySelectorAll('#rows .row.open').length;
  var text=r.querySelector('.detail p:nth-of-type(2)').textContent.slice(0,60);
  r.click();
  var closed=getComputedStyle(r.querySelector('.detail')).display;
  return JSON.stringify({before:before, after:after, closed:closed, openCount:openCount, note:text,
    expandable: document.querySelectorAll('#rows .row.has-detail').length});
})()
```

Expected: `before: "none"`, `after: "block"`, `closed: "none"`, `openCount: 1`, a real note string, and `expandable` close to 109 minus the unknowns.

Also confirm clicking an announcement link does NOT toggle the row (the `e.target.closest("a")` guard).

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "feat: expandable rows revealing the shutdown reason and key figures"
```

---

### Task 5: Reason filter

**Files:**
- Modify: `index.html` (controls markup, `state`, `apply`, event wiring)

- [ ] **Step 1: Add the select**

After the closure `<select>` block, add:

```html
          <div class="field">
            <select id="sel-why" aria-label="Filter by reason for shutdown"></select>
          </div>
```

- [ ] **Step 2: Populate it and extend state**

Change `var state = { q:"", cat:"", closure:"", month:"", sort:"date-desc" };` to include `why:""`.

Populate the select after `selCat` is populated:

```js
  var selWhy = el("sel-why");
  selWhy.innerHTML = '<option value="">All reasons</option>' +
    WHY_ORDER.filter(function (k){ return DATA.some(function (d){ return d.why === k; }); })
      .map(function (k){ return '<option value="' + k + '">' + esc(WHY_LABEL[k]) + '</option>'; }).join("");
```

- [ ] **Step 3: Filter on it**

In `apply()`, inside the filter callback, add:

```js
      if (state.why && d.why !== state.why) return false;
```

and in the result-line parts, add:

```js
    if (state.why) parts.push(esc(WHY_LABEL[state.why]));
```

Wire the listener:

```js
  selWhy.addEventListener("change", function (e){ state.why = e.target.value; apply(); });
```

- [ ] **Step 4: Verify the filter**

In the browser:

```js
(function(){
  var s=document.getElementById('sel-why');
  var total=document.querySelectorAll('#rows .row').length;
  s.value='exploit'; s.dispatchEvent(new Event('change',{bubbles:true}));
  var exploit=document.querySelectorAll('#rows .row').length;
  var allExploit=Array.from(document.querySelectorAll('#rows .why-chip')).every(c=>c.className.indexOf('why-exploit')>-1);
  var line=document.getElementById('resultline').textContent;
  // combine with category
  document.getElementById('sel-cat').value='DeFi';
  document.getElementById('sel-cat').dispatchEvent(new Event('change',{bubbles:true}));
  var combined=document.querySelectorAll('#rows .row').length;
  s.value=''; s.dispatchEvent(new Event('change',{bubbles:true}));
  document.getElementById('sel-cat').value=''; document.getElementById('sel-cat').dispatchEvent(new Event('change',{bubbles:true}));
  return JSON.stringify({total:total, exploit:exploit, allChipsMatch:allExploit, combined:combined,
    line:line, reset:document.querySelectorAll('#rows .row').length});
})()
```

Expected: `total: 109`, `exploit` > 0 and < 109, `allChipsMatch: true`, `combined` ≤ `exploit`, `reset: 109`.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "feat: filter the registry by shutdown reason"
```

---

### Task 6: "Why they died" chart

**Files:**
- Modify: `index.html` (charts markup, render code)

- [ ] **Step 1: Add the card**

In the charts grid, after the "How they announced it" card, add:

```html
        <div class="card">
          <h3>Why they died</h3>
          <p class="hint">The reason each shutdown was attributed to. Click a bar to filter.</p>
          <div class="hbars" id="why-bars"></div>
        </div>
```

- [ ] **Step 2: Compute and render**

Add `byWhy` alongside the existing tallies in the derived-stats block:

```js
  var byWhy = {};
  DATA.forEach(function (d) { var k = d.why || "unknown"; byWhy[k] = (byWhy[k]||0) + 1; });
```

Render it (mirroring the category bars, which are clickable):

```js
  var whyList = WHY_ORDER.filter(function (k){ return byWhy[k]; });
  var whyMax = Math.max.apply(null, whyList.map(function (k){ return byWhy[k]; }));
  el("why-bars").innerHTML = whyList.map(function (k, i) {
    var pct = Math.round((byWhy[k] / whyMax) * 100);
    return '<button class="hbar" data-why="' + esc(k) + '" style="all:unset;display:grid;grid-template-columns:116px 1fr 34px;align-items:center;gap:12px;cursor:pointer">' +
      '<span class="k">' + esc(WHY_SHORT[k]) + '</span>' +
      '<span class="track"><span class="fill" style="width:' + pct + '%;animation-delay:' + (i*60) + 'ms"></span></span>' +
      '<span class="n">' + byWhy[k] + '</span></button>';
  }).join("");
```

- [ ] **Step 3: Wire click-to-filter**

```js
  el("why-bars").addEventListener("click", function (e) {
    var b = e.target.closest("[data-why]"); if (!b) return;
    var k = b.dataset.why;
    state.why = (state.why === k) ? "" : k;
    selWhy.value = state.why;
    apply();
    document.getElementById("rg").scrollIntoView({ behavior: "smooth", block: "start" });
  });
```

- [ ] **Step 4: Verify the chart sums to 109**

```js
(function(){
  var vals=Array.from(document.querySelectorAll('#why-bars .n')).map(n=>+n.textContent);
  var sum=vals.reduce((a,b)=>a+b,0);
  document.querySelector('#why-bars [data-why]').click();
  var filtered=document.querySelectorAll('#rows .row').length;
  document.querySelector('#why-bars [data-why]').click();
  return JSON.stringify({bars:vals.length, sum:sum, clickFilters:filtered,
    afterToggleOff:document.querySelectorAll('#rows .row').length});
})()
```

Expected: `sum: 109`, `clickFilters` < 109, `afterToggleOff: 109`.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "feat: add the Why they died chart with click-to-filter"
```

---

### Task 7: Category × reason cross-reference

**Files:**
- Modify: `index.html` (charts markup, CSS, render code)

This is the analytically interesting view: it shows that DeFi dies to exploits while Gaming dies to funding.

- [ ] **Step 1: Add the card**

After the category card (which spans full width), add another full-width card:

```html
        <div class="card span2">
          <h3>What kills what</h3>
          <p class="hint">How each category died. Each row is one category; segment width is the share of that category's shutdowns.</p>
          <div id="cross"></div>
        </div>
```

- [ ] **Step 2: Add the CSS**

```css
  .xrow { display: grid; grid-template-columns: 116px 1fr 34px; align-items: center; gap: 12px; margin-bottom: 10px; }
  .xrow:last-child { margin-bottom: 0; }
  .xbar { display: flex; height: 16px; border: 1px solid var(--border); overflow: hidden; }
  .xseg { height: 100%; }
  .xlegend { display: flex; flex-wrap: wrap; gap: 12px; margin-top: 16px;
    padding-top: 14px; border-top: 1px solid var(--border); }
  .xlegend span { font-family: var(--font-mono); font-size: 11px; color: var(--dim);
    display: inline-flex; align-items: center; gap: 6px; }
  .xlegend i { width: 10px; height: 10px; display: inline-block; background: currentColor; }
```

- [ ] **Step 3: Render**

```js
  var WHY_HEX = { funding:"#d98c2b", exploit:"#d1566f", traction:"#8c93a1", market:"#6f9bd1",
    regulatory:"#b06fd1", insolvency:"#c0455c", acquired:"#4fae9b", dependency:"#c9a227",
    voluntary:"#7f9c6d", unknown:"#5c6672" };
  var crossCats = catList.slice();   // already sorted by count desc
  el("cross").innerHTML = crossCats.map(function (c) {
    var items = DATA.filter(function (d){ return d.cat === c; });
    var t = {}; items.forEach(function (d){ var k=d.why||"unknown"; t[k]=(t[k]||0)+1; });
    var segs = WHY_ORDER.filter(function (k){ return t[k]; }).map(function (k) {
      return '<span class="xseg" title="' + esc(WHY_LABEL[k]) + ': ' + t[k] + '" style="width:' +
        ((t[k]/items.length)*100) + '%;background:' + WHY_HEX[k] + '"></span>';
    }).join("");
    return '<div class="xrow"><span class="k">' + esc(c) + '</span>' +
      '<span class="xbar">' + segs + '</span><span class="n">' + items.length + '</span></div>';
  }).join("") +
  '<div class="xlegend">' + WHY_ORDER.filter(function (k){ return byWhy[k]; }).map(function (k) {
    return '<span style="color:' + WHY_HEX[k] + '"><i></i><em style="color:var(--dim);font-style:normal">' +
      esc(WHY_SHORT[k]) + '</em></span>';
  }).join("") + '</div>';
```

- [ ] **Step 4: Verify each row's segments total 100%**

```js
(function(){
  var rows=Array.from(document.querySelectorAll('.xrow'));
  var bad=rows.filter(function(r){
    var w=Array.from(r.querySelectorAll('.xseg')).reduce((a,s)=>a+parseFloat(s.style.width),0);
    return Math.abs(w-100)>0.5;
  }).length;
  var counts=rows.map(r=>+r.querySelector('.n').textContent).reduce((a,b)=>a+b,0);
  return JSON.stringify({rows:rows.length, rowsNotSummingTo100:bad, totalAcrossCategories:counts,
    legend:document.querySelectorAll('.xlegend span').length});
})()
```

Expected: `rowsNotSummingTo100: 0`, `totalAcrossCategories: 109`.

- [ ] **Step 5: Commit**

```bash
git add index.html && git commit -m "feat: add the category x reason cross-reference chart"
```

---

### Task 8: Mobile layout and the editorial-standard note

**Files:**
- Modify: `index.html` (mobile media query, Data & Contribute tab)

- [ ] **Step 1: Fold the chip and detail into mobile cards**

Inside the existing `@media (max-width: 760px)` block, add:

```css
    .c-why { padding-top: 2px; }
    .detail { padding: 14px 15px; }
    .xrow { grid-template-columns: 92px 1fr 30px; }
```

- [ ] **Step 2: Document the editorial standard**

In the Data & Contribute tab, inside the "Additions, verification & maintenance" card, add after the existing "How entries are sourced" paragraph:

```html
            <p><b>How the reason is decided.</b> A shutdown reason is an interpretation, not a hard fact — a project may say "strategic decision" when it in fact ran out of money. The tag reflects <b>what the source says</b>; the one-line note carries the nuance. Where no reason was stated anywhere, the entry is marked <b>Not reported</b> rather than guessed.</p>
```

- [ ] **Step 3: Verify mobile and the note**

Resize the browser to 375px wide and confirm cards render with the chip visible and rows still expand. Then:

```js
document.getElementById('view-data').innerHTML.indexOf('How the reason is decided') > -1
```

Expected: `true`.

- [ ] **Step 4: Commit**

```bash
git add index.html && git commit -m "feat: mobile layout for reason chip and document the editorial standard"
```

---

### Task 9: Update README and full verification

**Files:**
- Modify: `README.md`
- Verify: `index.html`

- [ ] **Step 1: Document the new fields for contributors**

In the contributing section of `README.md`, extend the entry-shape example:

```js
{ n:"Project Name",
  iso:"2026-07-25",
  day:true,
  url:"https://x.com/.../status/...",
  c:"x",              // x | docs | news | discord | deleted
  cat:"DeFi",
  why:"funding",      // funding | exploit | traction | market | regulatory
                      // insolvency | acquired | dependency | voluntary | unknown
  note:"Ran out of runway after failing to close a follow-on round.",
  figs:[{k:"Raised",v:"$5M"}] }
```

Add: "`why` must reflect what the source actually says. If no reason was stated, use `unknown` and omit `note`. Only include figures literally stated in a source."

- [ ] **Step 2: Run the full verification sweep**

```js
(function(){
  var rows=document.querySelectorAll('#rows .row');
  var sum=a=>a.reduce((x,y)=>x+y,0);
  var tl=sum(Array.from(document.querySelectorAll('#timeline .tl-bar .v')).map(v=>+v.textContent));
  var clos=sum(Array.from(document.querySelectorAll('#closure-bars .n')).map(v=>+v.textContent));
  var cat=sum(Array.from(document.querySelectorAll('#cat-bars .n')).map(v=>+v.textContent));
  var why=sum(Array.from(document.querySelectorAll('#why-bars .n')).map(v=>+v.textContent));
  return JSON.stringify({rows:rows.length, timeline:tl, closure:clos, category:cat, why:why,
    pageOverflow: document.documentElement.scrollWidth-document.documentElement.clientWidth,
    chips: document.querySelectorAll('.why-chip').length});
})()
```

Expected: every one of `rows`, `timeline`, `closure`, `category`, `why` equals **109**; `pageOverflow: 0`.

- [ ] **Step 3: Check both themes and the console**

Confirm no console errors. Toggle to light theme and confirm the reason chips and cross-reference segments stay legible against the light ground.

- [ ] **Step 4: Spot-check sourcing integrity**

Pick five entries with figures. For each, open its `url` and confirm the figure and the note actually match what the source says. Any that do not match must be corrected or the figure dropped.

- [ ] **Step 5: Commit and deploy**

```bash
git add README.md index.html
git commit -m "docs: document reason fields for contributors"
git push origin main
```

Then confirm the live site serves it:

```bash
for i in $(seq 1 25); do
  n=$(curl -s https://lainncalvo.github.io/signal-lost/index.html | grep -o 'why:"' | wc -l | tr -d ' ')
  if [ "$n" = "109" ]; then echo "DEPLOYED"; break; fi; sleep 6
done
```

Expected: `DEPLOYED`.

---

## Self-Review

**Spec coverage:** Data model → Task 1. Taxonomy → Task 2. Research → prerequisite, consolidated in Task 1. Editorial standard → Task 8 Step 2. Why column → Task 3. Row expansion → Task 4. Filter → Task 5. Charts (reason + cross-reference) → Tasks 6 and 7. Mobile → Task 8. Verification list → Task 9 plus per-task checks. No gaps.

**Placeholders:** none — every step carries its actual code or command.

**Type consistency:** `WHY_LABEL`, `WHY_SHORT`, `WHY_ORDER`, `WHY_HEX`, `byWhy`, `state.why`, `selWhy`, and the `why`/`note`/`figs` fields are used identically across Tasks 2–9. `catList` referenced in Task 7 is the existing variable defined in the category-bars block.

**Known risk:** Task 3 Step 4 gates on the six-column layout fitting. The fallback is specified inline so the implementer is never blocked.
