<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Fridge-to-Dinner</title>
<style>
:root{--bg:#fbf8f3;--card:#fff;--text:#1f2a24;--muted:#6b756f;--accent:#2f8f5b;--accent-ink:#fff;--line:#e6e0d6;--warn:#c2571a;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px);color-scheme:light}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#141a16;--card:#1d2620;--text:#eef3ef;--muted:#9aa89f;--accent:#4cc585;--accent-ink:#0d1f15;--line:#2d3a31;--warn:#f09a63;color-scheme:dark}}
:root[data-theme="dark"]{--bg:#141a16;--card:#1d2620;--text:#eef3ef;--muted:#9aa89f;--accent:#4cc585;--accent-ink:#0d1f15;--line:#2d3a31;--warn:#f09a63;color-scheme:dark}
*{box-sizing:border-box}
html{scroll-padding-top:env(safe-area-inset-top,0px)}
body{margin:0;background:var(--bg);color:var(--text);font:16px/1.5 system-ui,-apple-system,"Segoe UI",Roboto,sans-serif}
main{max-width:760px;margin:0 auto;padding:24px 16px 56px}
h1{font-size:1.9rem;margin:0 0 4px}
.sub{color:var(--muted);margin:0 0 20px}
.card{background:var(--card);border:1px solid var(--line);border-radius:16px;padding:16px;margin-bottom:16px}
.drop{display:flex;flex-direction:column;align-items:center;gap:10px;text-align:center;border:2px dashed var(--line);border-radius:16px;padding:24px;background:var(--card)}
.drop img{max-width:100%;max-height:260px;border-radius:12px}
.row{display:flex;gap:8px;flex-wrap:wrap;justify-content:center}
button,label.btn{font:inherit;border:0;border-radius:999px;padding:10px 18px;cursor:pointer;background:var(--accent);color:var(--accent-ink);font-weight:600}
button.ghost,label.btn.ghost{background:transparent;color:var(--text);border:1px solid var(--line)}
button:disabled{opacity:.5;cursor:default}
input[type=file]{display:none}
.chips{display:flex;flex-wrap:wrap;gap:6px}
.chip{background:var(--bg);border:1px solid var(--line);border-radius:999px;padding:3px 12px;font-size:.9rem}
.chip button{all:unset;cursor:pointer;margin-left:6px;color:var(--muted)}
.add{display:flex;gap:8px;margin-top:10px}
.add input{flex:1;font:inherit;padding:8px 12px;border-radius:999px;border:1px solid var(--line);background:var(--bg);color:var(--text)}
h2{font-size:1.15rem;margin:0 0 8px}
.meta{color:var(--muted);font-size:.88rem;margin-bottom:8px}
.recipe h3{margin:0 0 2px}
.cols{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px}
.miss{color:var(--warn)}
ul,ol{margin:6px 0;padding-left:20px}
.status{color:var(--muted);text-align:center;padding:12px}
.err{color:var(--warn)}
.list label{display:flex;gap:8px;align-items:center;padding:4px 0}
.small{font-size:.82rem;color:var(--muted);text-align:center}
</style>
</head>
<body>
<main>
  <h1>🥘 Fridge-to-Dinner</h1>
  <p class="sub">Snap your fridge or pantry. Get dinner ideas and a shopping list for what’s missing.</p>

  <div class="drop" id="drop">
    <div id="preview">No photo yet</div>
    <div class="row">
      <label class="btn">📷 Take or choose photo<input type="file" id="file" accept="image/*" capture="environment"></label>
      <button class="ghost" id="demo">Try demo mode</button>
    </div>
  </div>

  <div id="status" class="status" hidden></div>

  <section id="ing" class="card" hidden>
    <h2>What I found</h2>
    <div class="chips" id="chips"></div>
    <div class="add"><input id="newIng" placeholder="Add something I missed"><button class="ghost" id="addBtn">Add</button></div>
    <div class="row" style="margin-top:12px"><button id="again">Suggest recipes</button></div>
  </section>

  <section id="out" hidden></section>
  <p class="small">AI suggestions can be wrong. Check ingredients and allergies yourself.</p>
</main>

<script>
(function(){
  var $=function(id){return document.getElementById(id)};
  var state={ings:[],sample:null,photo:null};
  var DEMO={ingredients:["eggs","spinach","cheddar cheese","tomatoes","leftover rice","milk","butter","onion","bell pepper","tortillas"],
  recipes:[
   {name:"Veggie Fried Rice",time:"20 min",uses:["leftover rice","eggs","onion","bell pepper","spinach"],missing:["soy sauce","sesame oil"],steps:["Sauté diced onion and pepper in butter.","Add rice and fry until toasty.","Push aside, scramble eggs, then mix in spinach.","Season with soy sauce and finish with sesame oil."]},
   {name:"Spinach Cheddar Quesadillas",time:"15 min",uses:["tortillas","cheddar cheese","spinach","tomatoes"],missing:["salsa"],steps:["Layer cheese, spinach and tomato on a tortilla.","Fold and toast in a buttered pan until crisp.","Slice and serve with salsa."]},
   {name:"Garden Frittata",time:"25 min",uses:["eggs","milk","cheddar cheese","onion","bell pepper","tomatoes"],missing:["fresh herbs"],steps:["Whisk eggs with milk and cheese.","Soften onion and pepper in an oven-safe pan.","Pour in eggs, add tomatoes, cook 3 min.","Bake at 200°C for 10 min until set."]}
  ]};

  function status(msg,err){var s=$("status");s.hidden=!msg;s.textContent=msg||"";s.className="status"+(err?" err":"")}
  function esc(t){var d=document.createElement("div");d.textContent=t;return d.innerHTML}

  function shrink(file){return new Promise(function(res,rej){
    var img=new Image(),url=URL.createObjectURL(file);
    img.onload=function(){
      var m=1280,r=Math.min(1,m/Math.max(img.width,img.height));
      var c=document.createElement("canvas");c.width=Math.round(img.width*r);c.height=Math.round(img.height*r);
      c.getContext("2d").drawImage(img,0,0,c.width,c.height);
      c.toBlob(function(b){URL.revokeObjectURL(url);b?res(b):rej(new Error("Could not read image"))},"image/jpeg",0.85)};
    img.onerror=function(){rej(new Error("Could not read image"))};img.src=url})}

  function parse(t){
    var s=t.replace(/```json|```/g,"").trim();
    var a=s.indexOf("{"),b=s.lastIndexOf("}");
    return JSON.parse(s.slice(a,b+1))}

  function renderChips(){
    var c=$("chips");c.innerHTML="";
    state.ings.forEach(function(x,i){
      var s=document.createElement("span");s.className="chip";s.textContent=x;
      var b=document.createElement("button");b.textContent="×";b.setAttribute("aria-label","Remove "+x);
      b.onclick=function(){state.ings.splice(i,1);renderChips()};
      s.appendChild(b);c.appendChild(s)});
    $("ing").hidden=false}

  function renderRecipes(list){
    var out=$("out"),shop={};
    var h='<div class="card"><h2>Dinner ideas</h2></div>';
    list.forEach(function(r){
      (r.missing||[]).forEach(function(m){shop[m]=1});
      h+='<div class="card recipe"><h3>'+esc(r.name)+'</h3><div class="meta">'+esc(r.time||"")+'</div>'
        +'<div class="cols"><div><strong>You have</strong><ul>'+(r.uses||[]).map(function(u){return"<li>"+esc(u)+"</li>"}).join("")+'</ul>'
        +'<strong class="miss">Need</strong><ul>'+((r.missing||[]).length?r.missing.map(function(u){return"<li>"+esc(u)+"</li>"}).join(""):"<li>Nothing, you’re set!</li>")+'</ul></div>'
        +'<div><strong>Steps</strong><ol>'+(r.steps||[]).map(function(u){return"<li>"+esc(u)+"</li>"}).join("")+'</ol></div></div></div>'});
    var items=Object.keys(shop);
    h+='<div class="card list"><h2>🛒 Shopping list</h2>'+(items.length?items.map(function(m){return'<label><input type="checkbox"> '+esc(m)+'</label>'}).join(""):"<p>You have everything!</p>")+'</div>';
    out.innerHTML=h;out.hidden=false;out.scrollIntoView({behavior:"smooth"})}

  async function ask(prompt,images){
    if(!state.sample)throw new Error("AI isn’t available here. Try demo mode.");
    var opts={cache:false};if(images)opts.images=images;
    var r=await state.sample(prompt,opts);
    return parse(r.text)}

  async function analyze(file){
    try{
      status("Looking at your photo…");$("out").hidden=true;
      var blob=await shrink(file);
      $("preview").innerHTML='<img alt="Your photo" src="'+URL.createObjectURL(blob)+'">';
      var lim=state.sample&&state.sample.limits?await state.sample.limits():{images:true};
      if(!lim.images)throw new Error("Image analysis isn’t available for this viewer. Try demo mode.");
      var data=await ask('List the food ingredients visible in this fridge or pantry photo. Respond with ONLY JSON: {"ingredients":["..."]}. Use short common names, no brands, max 25 items.',[blob]);
      state.ings=(data.ingredients||[]).map(String);
      status("");renderChips();
      if(!state.ings.length)status("I couldn’t spot any food. Try a closer, brighter photo.",true);
      else await suggest()
    }catch(e){status((e&&e.message)||"Something went wrong. Try demo mode.",true)}}

  async function suggest(){
    try{
      if(!state.ings.length){status("Add at least one ingredient first.",true);return}
      status("Cooking up ideas…");$("again").disabled=true;
      var data=await ask('I have these ingredients: '+state.ings.join(", ")+'. Suggest 3 realistic dinners, favoring ones that use mostly what I have (assume salt, pepper and cooking oil). Respond with ONLY JSON: {"recipes":[{"name":"","time":"","uses":["from my list"],"missing":["items I must buy"],"steps":["4-6 short steps"]}]}');
      status("");renderRecipes(data.recipes||[])
    }catch(e){status((e&&e.message)||"Couldn’t get recipes. Try demo mode.",true)}
    finally{$("again").disabled=false}}

  $("file").onchange=function(e){var f=e.target.files[0];if(f)analyze(f)};
  $("demo").onclick=function(){
    $("preview").textContent="Demo mode: sample fridge";
    state.ings=DEMO.ingredients.slice();status("");renderChips();renderRecipes(DEMO.recipes)};
  $("again").onclick=suggest;
  function addIng(){var v=$("newIng").value.trim();if(v){state.ings.push(v);$("newIng").value="";renderChips()}}
  $("addBtn").onclick=addIng;
  $("newIng").onkeydown=function(e){if(e.key==="Enter")addIng()};

  if(window.claude&&claude.use){claude.use("sample").then(function(s){state.sample=s}).catch(function(){})}
})();
</script>
</body>
</html>
