<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Tracker de pêche — Carte interactive</title>
  <style>
    :root{--bg:#0f172a;--card:#0b1220;--accent:#60a5fa;--muted:#94a3b8}
    *{box-sizing:border-box}
    body{font-family:Inter,Segoe UI,Arial,sans-serif;margin:0;background:linear-gradient(180deg,var(--bg),#071029);color:#e6eef8}
    .wrap{max-width:1100px;margin:32px auto;padding:24px}
    header{display:flex;align-items:center;gap:16px}
    h1{margin:0;font-size:1.6rem}
    p.lead{margin:6px 0 18px;color:var(--muted)}

    .main{display:grid;grid-template-columns:1fr 360px;gap:20px}
    .map-card{background:linear-gradient(180deg,#071029 0%, #0b1220 100%);border-radius:12px;padding:12px;box-shadow:0 6px 18px rgba(2,6,23,.6)}

    /* zone carte */
    .map{position:relative;width:100%;height:640px;border-radius:8px;overflow:hidden;background-image:linear-gradient(180deg, rgba(96,165,250,0.03), rgba(147,51,234,0.03));display:flex;align-items:center;justify-content:center}
    .map img.background{width:100%;height:100%;object-fit:cover;filter:brightness(.85)}

    /* marqueurs de poissons */
    .marker{position:absolute;transform:translate(-50%,-50%);cursor:pointer;transition:transform .12s ease}
    .marker:hover{transform:translate(-50%,-50%) scale(1.12)}
    .marker .dot{width:18px;height:18px;border-radius:50%;background:var(--accent);display:flex;align-items:center;justify-content:center;color:#022; font-weight:700}
    .marker.caught .dot{background:linear-gradient(90deg,#10b981,#3b82f6)}

    /* carte responsive */
    @media (max-width:980px){.main{grid-template-columns:1fr;}.map{height:520px}.sidebar{order:2}}

    /* sidebar */
    .sidebar{background:linear-gradient(180deg,#071233,#071026);padding:18px;border-radius:12px;height:640px;overflow:auto}
    .fish-item{display:flex;align-items:center;gap:12px;padding:8px;border-radius:8px;margin-bottom:8px}
    .fish-item .icon{width:44px;height:44px;border-radius:8px;background:#081428;display:flex;align-items:center;justify-content:center}
    .fish-name{font-weight:600}
    .meta{color:var(--muted);font-size:.9rem}
    .fish-item.caught{opacity:.6;text-decoration:line-through}

    .controls{display:flex;gap:8px;margin-top:12px}
    button{background:#0ea5a1;border:none;padding:8px 12px;border-radius:8px;color:white;cursor:pointer}
    button.ghost{background:transparent;border:1px solid rgba(255,255,255,.06)}
    .small{padding:6px 8px;font-size:.9rem}

    /* tooltip */
    .tooltip{position:absolute;background:#021026;padding:8px 10px;border-radius:8px;border:1px solid rgba(255,255,255,.04);font-size:.9rem;pointer-events:none;transform:translate(-50%,-120%);white-space:nowrap}

    footer{margin-top:18px;color:var(--muted);font-size:.9rem}
  </style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Tracker de pêche — Carte interactive</h1>
    </header>
    <p class="lead">Glisse la souris sur un poisson pour voir où le capturer. Clique dessus pour marquer comme attrapé. Le progrès est sauvegardé localement.</p>

    <div class="main">
      <div class="map-card">
        <div id="map" class="map" aria-label="Carte de pêche interactive">
          <!-- Remplace l'image background par la carte de ton jeu (mettre un fichier dans images/ ou URL externe) -->
          <img class="background" src="https://images.unsplash.com/photo-1507525428034-b723cf961d3e?q=80&w=1600&auto=format&fit=crop&ixlib=rb-4.0.3&s=4a1f6d0cc2a7a2b1c9f4f6b4c1d9b6d3" alt="Carte de la région de pêche"/>
        </div>
      </div>

      <aside class="sidebar">
        <h3>Liste des poissons</h3>
        <div id="list"></div>

        <div class="controls">
          <button id="reset" class="small ghost">Réinitialiser</button>
          <button id="export" class="small">Exporter</button>
          <button id="importBtn" class="small ghost">Importer</button>
          <input id="import" type="file" accept="application/json" style="display:none" />
        </div>

        <footer>
          <p>Astuce : tu peux remplacer l'image de fond par la carte de ton jeu. Modifie la variable <code>mapBg</code> dans le fichier.</p>
        </footer>
      </aside>
    </div>
  </div>

  <script>
    // ----- CONFIGURATION -----
    // Liste des poissons : id, nom, description, coords (pourcentage x,y), rareté
    const fishData = [
      {id:'f1', name:'Truite argentée', desc:'Près des rapides, profondeur moyenne.', x:22, y:40, rarity:'commune'},
      {id:'f2', name:'Brochet véloci', desc:'Bordure des roseaux.', x:68, y:52, rarity:'rare'},
      {id:'f3', name:'Carassin doré', desc:'Petites baies calmes.', x:44, y:74, rarity:'commune'},
      {id:'f4', name:'Lucioperca', desc:'Zones rocheuses profondes.', x:80, y:22, rarity:'epique'},
      {id:'f5', name:'Poisson-lune', desc:'Lac central la nuit.', x:50, y:36, rarity:'legendary'}
    ];

    const map = document.getElementById('map');
    const listEl = document.getElementById('list');
    const STORAGE_KEY = 'fish-tracker-v1';

    // récupère l'état depuis localStorage
    function loadState(){
      try{
        const raw = localStorage.getItem(STORAGE_KEY);
        return raw ? JSON.parse(raw) : {};
      }catch(e){return {}};
    }
    function saveState(state){localStorage.setItem(STORAGE_KEY, JSON.stringify(state))}

    let state = loadState(); // { f1: true, f2: false }

    // crée les marqueurs sur la carte
    function createMarkers(){
      fishData.forEach(f=>{
        const el = document.createElement('div');
        el.className = 'marker' + (state[f.id] ? ' caught' : '');
        el.dataset.id = f.id;
        el.style.left = f.x + '%';
        el.style.top = f.y + '%';

        el.innerHTML = `<div class="dot" title="${escapeHtml(f.name)}">${state[f.id] ? '✓' : '●'}</div>`;

        // tooltip element
        const tip = document.createElement('div');
        tip.className = 'tooltip';
        tip.style.display = 'none';
        tip.innerHTML = `<strong>${escapeHtml(f.name)}</strong><div class='meta'>${escapeHtml(f.desc)}</div><div class='meta'>Rareté: ${f.rarity}</div>`;
        el.appendChild(tip);

        // hover handlers
        el.addEventListener('mouseenter', ()=>{ tip.style.display='block'; });
        el.addEventListener('mouseleave', ()=>{ tip.style.display='none'; });

        // click -> toggle caught
        el.addEventListener('click', (ev)=>{
          ev.stopPropagation();
          toggleCaught(f.id);
        });

        map.appendChild(el);
      });
    }

    // crée la liste latérale
    function renderList(){
      listEl.innerHTML = '';
      fishData.forEach(f=>{
        const item = document.createElement('div');
        item.className = 'fish-item' + (state[f.id] ? ' caught' : '');
        item.dataset.id = f.id;

        item.innerHTML = `<div class='icon'>${state[f.id] ? '✓' : '🎣'}</div>
                          <div>
                            <div class='fish-name'>${escapeHtml(f.name)}</div>
                            <div class='meta'>${escapeHtml(f.desc)}</div>
                          </div>`;

        item.addEventListener('click', ()=> toggleCaught(f.id));
        listEl.appendChild(item);
      });
    }

    function toggleCaught(id){
      state[id] = !state[id];
      saveState(state);
      // refresh markers and list
      refreshUI();
    }

    function refreshUI(){
      // markers
      document.querySelectorAll('.marker').forEach(m=>{
        const id = m.dataset.id;
        if(state[id]){
          m.classList.add('caught');
          m.querySelector('.dot').textContent = '✓';
        } else {
          m.classList.remove('caught');
          m.querySelector('.dot').textContent = '●';
        }
      });
      // list
      renderList();
    }

    // reset
    document.getElementById('reset').addEventListener('click', ()=>{
      if(!confirm('Réinitialiser ton suivi ?')) return;
      state = {};
      saveState(state);
      refreshUI();
    });

    // export
    document.getElementById('export').addEventListener('click', ()=>{
      const blob = new Blob([JSON.stringify(state, null, 2)], {type:'application/json'});
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a'); a.href=url; a.download='fish-tracker.json'; a.click(); URL.revokeObjectURL(url);
    });

    // import
    const importInput = document.getElementById('import');
    document.getElementById('importBtn').addEventListener('click', ()=> importInput.click());
    importInput.addEventListener('change', async (e)=>{
      const file = e.target.files[0]; if(!file) return;
      try{
        const txt = await file.text();
        const obj = JSON.parse(txt);
        state = obj;
        saveState(state);
        refreshUI();
        alert('Importation OK');
      }catch(err){alert('Fichier invalide')}
    });

    // helper pour échapper le HTML
    function escapeHtml(s){return String(s).replace(/[&<>\"]/g, c=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[c]))}

    // initialisation
    createMarkers();
    renderList();

    // click pour fermer tooltip si clic hors marker
    map.addEventListener('click', ()=> document.querySelectorAll('.tooltip').forEach(t=>t.style.display='none'));

    // conseil : pour remplacer l'image, modifie la balise <img class="background" src="...">
  </script>
</body>
</html>
