
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Sistema Solar — Transição em Movimento</title>
  <style>
    :root{
      --bg:#030414;
      --stars:#0b1220;
      --sun-color: #ffcc33;
      --accent: rgba(255,255,255,0.06);
    }
    *{box-sizing:border-box}
    html,body{height:100%;margin:0;font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial}
    body{
      background: radial-gradient(1200px 800px at 10% 10%, rgba(255,255,255,0.02), transparent),
                  linear-gradient(180deg,var(--bg), #061126 80%);
      color:#e6eef8;overflow:hidden;
      display:flex;align-items:center;justify-content:center;
    }

    /* área principal com viewBox virtual */
    .space {
      width:100vw;height:100vh;position:relative;overflow:hidden;
      -webkit-tap-highlight-color: transparent;
    }

    /* Estrelas de fundo */
    .stars{position:absolute;inset:0;background-image:radial-gradient(circle at 20% 30%, rgba(255,255,255,0.06) 0 1px, transparent 1px),
      radial-gradient(circle at 70% 80%, rgba(255,255,255,0.04) 0 1px, transparent 1px);
      background-size: 200px 200px, 300px 300px;filter:blur(.6px)}

    /* cena central que pode ser transicionada (zoom/pan) */
    .scene{
      position:absolute;left:50%;top:50%;transform:translate(-50%,-50%) scale(1);
      width:1200px;height:800px;max-width:1200px;max-height:800px;pointer-events:none;
      transition:transform 1s cubic-bezier(.22,.9,.12,1);
    }

    /* Sol central */
    .sun{
      position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);
      width:120px;height:120px;border-radius:50%;
      background: radial-gradient(circle at 30% 30%, #fff6d6 0%, var(--sun-color) 40%, #ff9900 65%, #bb5f00 100%);
      box-shadow: 0 0 60px rgba(255,200,80,0.35), 0 0 120px rgba(255,160,60,0.12) inset;
      z-index:40;display:flex;align-items:center;justify-content:center;pointer-events:auto;
    }

    .sun::after{content:'';position:absolute;inset:-80px;border-radius:50%;filter:blur(40px);opacity:.25}

    /* órbitas */
    .orbit{
      position:absolute;left:50%;top:50%;transform:translate(-50%,-50%);border-radius:50%;
      border:1px dashed rgba(255,255,255,0.04);pointer-events:none;
    }

    .planet{
      position:absolute;left:50%;top:50%;transform-origin:0 -1px; /* rota ao redor do centro */
      will-change:transform;pointer-events:auto;
    }

    /* dicas e controles */
    .hud{position:absolute;left:18px;top:18px;font-size:13px;opacity:.9}
    .controls{position:absolute;right:18px;top:18px;display:flex;gap:8px}
    .btn{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.06);padding:8px 12px;border-radius:8px;color:#fff;cursor:pointer}
    .btn:active{transform:translateY(1px)}

    /* legendas pequenas dos planetas */
    .label{position:absolute;font-size:12px;padding:4px 6px;border-radius:6px;background:rgba(0,0,0,0.35);backdrop-filter:blur(4px)}

    /* animação fluida (será controlada por JS) */
    .animating{pointer-events:auto}

    /* responsividade */
    @media (max-width:900px){
      .scene{width:1000px;height:666px}
      .sun{width:90px;height:90px}
    }
    @media (max-width:520px){
      .hud,.controls{font-size:12px}
    }
  </style>
</head>
<body>
  <div class="space" id="space" aria-label="Transição do sistema solar">
    <div class="stars" aria-hidden="true"></div>

    <div class="scene" id="scene" role="img" aria-label="Sistema Solar em movimento">
      <div class="sun" id="sun" title="Sol"></div>
      <!-- órbitas e planetas serão gerados por javascript -->
    </div>

    <div class="hud">Sistema Solar — Clique nos planetas para focalizar / Pressione Espaço para pausar</div>
    <div class="controls">
      <button class="btn" id="toggleAnim">Pausar</button>
      <button class="btn" id="reset">Reset</button>
    </div>
  </div>

  <script>
    (function(){
      const scene = document.getElementById('scene');
      const sun = document.getElementById('sun');
      const toggleBtn = document.getElementById('toggleAnim');
      const resetBtn = document.getElementById('reset');

      // Definição simples dos planetas (nome, raio visual, órbita em px, período relativo)
      const planets = [
        {name:'Mercúrio', size:8, orbit:90, period:4.8, color:'#b5b5b5'},
        {name:'Vênus',    size:12, orbit:140, period:12.2, color:'#e3c07a'},
        {name:'Terra',    size:13, orbit:190, period:16, color:'#4aa3ff', hasMoon:true},
        {name:'Marte',    size:10, orbit:240, period:30, color:'#d86b4f'},
        {name:'Júpiter',  size:28, orbit:320, period:95, color:'#d9b48a'},
        {name:'Saturno',  size:24, orbit:410, period:120, color:'#e6cf9f', ring:true},
        {name:'Urano',    size:18, orbit:490, period:240, color:'#8fe0e8'},
        {name:'Netuno',   size:18, orbit:560, period:400, color:'#3e6cff'}
      ];

      const state = { running:true, time:0, zoom:1, focused:null };

      // criar órbitas e planetas DOM
      planets.forEach((p, i) => {
        // órbita
        const orbit = document.createElement('div');
        orbit.className = 'orbit';
        orbit.style.width = orbit.style.height = (p.orbit*2) + 'px';
        orbit.style.marginLeft = -(p.orbit) + 'px';
        orbit.style.marginTop = -(p.orbit) + 'px';
        orbit.style.zIndex = 10 + i;
        scene.appendChild(orbit);

        // planeta wrapper (para rotacionar)
        const wrapper = document.createElement('div');
        wrapper.className = 'planet';
        wrapper.style.zIndex = 60 + i;

        // elemento do planeta (circular)
        const el = document.createElement('div');
        el.className = 'planet-body';
        el.style.width = el.style.height = p.size + 'px';
        el.style.borderRadius = '50%';
        el.style.background = p.color;
        el.style.position = 'absolute';
        // posicionar sobre a borda da órbita (à direita)
        el.style.left = (scene.clientWidth/2 + p.orbit) + 'px';
        el.style.top = (scene.clientHeight/2 - (p.size/2)) + 'px';
        el.style.margin = '0';
        el.style.boxShadow = '0 4px 14px rgba(0,0,0,.35)';
        el.style.cursor = 'pointer';

        // rótulo
        const label = document.createElement('div');
        label.className = 'label';
        label.textContent = p.name;
        label.style.left = (scene.clientWidth/2 + p.orbit + p.size + 8) + 'px';
        label.style.top = (scene.clientHeight/2 - 10) + 'px';

        wrapper.appendChild(el);
        scene.appendChild(wrapper);
        scene.appendChild(label);

        // anota referências
        p.dom = {orbit, wrapper, el, label};

        // eventos
        el.addEventListener('click', (ev) => {
          ev.stopPropagation();
          focusPlanet(p);
        });
      });

      // ajustar posições responsivas quando redimensionar
      window.addEventListener('resize', () => {
        planets.forEach(p => {
          const el = p.dom.el;
          el.style.left = (scene.clientWidth/2 + p.orbit) + 'px';
          el.style.top = (scene.clientHeight/2 - (p.size/2)) + 'px';
          p.dom.label.style.left = (scene.clientWidth/2 + p.orbit + p.size + 8) + 'px';
          p.dom.label.style.top = (scene.clientHeight/2 - 10) + 'px';
        });
      });

      // foco em planeta — faz transição (zoom/pan) para aproximar o planeta
      function focusPlanet(p){
        if(!p) return resetFocus();
        state.focused = p;
        const rect = scene.getBoundingClientRect();
        const centerX = rect.left + rect.width/2;
        const centerY = rect.top + rect.height/2;

        // calcular posição do planeta atual em coords da cena
        const angle = (state.time / p.period) * 2 * Math.PI;
        const px = rect.left + rect.width/2 + Math.cos(angle) * p.orbit;
        const py = rect.top + rect.height/2 + Math.sin(angle) * p.orbit;

        // diferença para centralizar
        const dx = centerX - px;
        const dy = centerY - py;

        // ajustar transform: translate + scale
        const scale = Math.min(2.8, 600 / (p.size + 60));
        scene.style.transition = 'transform 900ms cubic-bezier(.22,.9,.12,1)';
        scene.style.transform = `translate(calc(-50% + ${dx}px), calc(-50% + ${dy}px)) scale(${scale})`;

        // realçar planeta
        planets.forEach(q => q.dom.el.style.filter = (q===p) ? 'brightness(1.25) drop-shadow(0 8px 18px rgba(0,0,0,.5))' : 'brightness(.6)');
      }

      function resetFocus(){
        state.focused = null;
        scene.style.transition = 'transform 900ms cubic-bezier(.22,.9,.12,1)';
        scene.style.transform = 'translate(-50%,-50%) scale(1)';
        planets.forEach(q => q.dom.el.style.filter = '');
      }

      // animação principal: atualiza posições com base no tempo
      function tick(dt){
        if(!state.running) return;
        state.time += dt * 0.02; // controle da velocidade global

        planets.forEach(p => {
          const angle = (state.time / p.period) * 2 * Math.PI;
          // atualizar wrapper rotacionando ao redor do sol
          const x = Math.cos(angle) * p.orbit;
          const y = Math.sin(angle) * p.orbit;
          p.dom.wrapper.style.transform = `translate(${scene.clientWidth/2}px, ${scene.clientHeight/2}px) translate(${x}px, ${y}px)`;
          // manter label próximo
          p.dom.label.style.left = (scene.clientWidth/2 + x + p.size + 8) + 'px';
          p.dom.label.style.top = (scene.clientHeight/2 + y - 10) + 'px';

          // aninhar lua para a Terra como exemplo
          if(p.hasMoon){
            if(!p.moon){
              const moon = document.createElement('div');
              moon.style.width = moon.style.height = '6px';
              moon.style.borderRadius = '50%';
              moon.style.background = '#dcdcdc';
              moon.style.position = 'absolute';
              moon.style.left = (scene.clientWidth/2 + p.orbit + 18) + 'px';
              moon.style.top = (scene.clientHeight/2 - 4) + 'px';
              moon.style.boxShadow = '0 4px 8px rgba(0,0,0,.4)';
              scene.appendChild(moon);
              p.moon = moon;
            }
            const moonAng = angle * 12; // lua gira mais rápido ao redor da Terra
            const mx = Math.cos(moonAng) * 22;
            const my = Math.sin(moonAng) * 22;
            p.moon.style.left = (scene.clientWidth/2 + Math.cos(angle) * p.orbit + mx) + 'px';
            p.moon.style.top = (scene.clientHeight/2 + Math.sin(angle) * p.orbit + my) + 'px';
          }
        });

        // se focado, atualiza pan para manter foco no planeta em movimento
        if(state.focused){
          focusPlanet(state.focused);
        }
      }

      // loop usando requestAnimationFrame
      let last = performance.now();
      function loop(now){
        const dt = now - last; last = now;
        tick(dt);
        requestAnimationFrame(loop);
      }
      requestAnimationFrame(loop);

      // controles
      toggleBtn.addEventListener('click', () =>{
        state.running = !state.running;
        toggleBtn.textContent = state.running ? 'Pausar' : 'Executar';
      });

      resetBtn.addEventListener('click', ()=>{
        resetFocus();
        state.time = 0;
      });

      // atalho de teclado: espaço pausa/retoma, esc reseta foco
      window.addEventListener('keydown', (e)=>{
        if(e.code === 'Space'){
          e.preventDefault();
          state.running = !state.running;
          toggleBtn.textContent = state.running ? 'Pausar' : 'Executar';
        }else if(e.code === 'Escape'){
          resetFocus();
        }
      });

      // clique no background reseta foco
      document.getElementById('space').addEventListener('click', ()=> resetFocus());

      // acessibilidade: reduz animação se preferido
      if(window.matchMedia('(prefers-reduced-motion: reduce)').matches){
        state.running = false; toggleBtn.textContent = 'Executar';
      }

    })();
  </script>
</body>
</html>
