# Gêneros & Mecânicas-Núcleo

Anatomia de cada gênero pixel comum: o que o define, sistemas essenciais e como
estruturá-los. Use para decidir escopo e saber o que implementar.

---

## 1. Plataforma (platformer / metroidvania)

**Núcleo:** movimento preciso + pulo + colisão com tiles.

Sistemas essenciais:
- **Controller responsivo** com aceleração/atrito, *coyote time* (pular logo
  após sair da plataforma), *jump buffer* (registrar pulo apertado antes de
  tocar o chão) e **pulo variável** (soltar cedo = pulo mais baixo). Esses três
  detalhes são o que faz "parecer bom" (ver código em `engines-frameworks.md`).
- **Colisão AABB com tiles**, resolvida por eixo (mover X, resolver; mover Y,
  resolver) para evitar travamentos em cantos.
- **Câmera** com look-ahead (antecipa a direção) e limites de fase.
- **Hazards e checkpoints**, plataformas móveis, escadas, one-way platforms
  (sobe por baixo, pisa por cima).
- **Metroidvania:** adicione mapa interconectado, habilidades que destravam
  áreas (double jump, dash, wall-jump) e backtracking. Salve o estado de portas/
  upgrades.

Loop mínimo jogável: player que corre/pula numa fase Tiled com inimigo e meta.

---

## 2. RPG top-down (estilo clássico / Zelda-like / turn-based)

**Núcleo:** mundo explorável + NPCs/diálogo + combate + progressão + inventário.

Sistemas essenciais:
- **Movimento top-down** (4 ou 8 direções), com `AnimationTree`/blend por
  direção para virar o sprite.
- **Mapas conectados** com transições (sair de uma sala entra na próxima),
  triggers e portas.
- **NPCs + diálogo ramificado** (ver `narrative-dialogue.md`).
- **Inventário e itens:** estrutura de dados de item (id, tipo, stack), UI de
  grid, equipar/usar.
- **Combate:**
  - *Action* (Zelda-like): hitboxes/hurtboxes em tempo real, i-frames após dano.
  - *Turn-based:* fila de turnos por velocidade, ações (atacar/skill/item/fugir),
    cálculo de dano (atk vs def + variação), estados (veneno, atordoar), e uma
    máquina de estados de batalha (seleção → resolução → checagem de fim).
- **Progressão:** XP, level-up, stats, árvore/equipamento.
- **Save/load:** serialize posição, inventário, flags de quest, stats.

Zelda-likes se beneficiam de engines de gênero pronto (Solarus, em Lua, já traz
herói, mapas, inimigos, HUD e itens). RPGs turn-based clássicos têm estrutura
de "open RPG" reaproveitável: grid de mundo, party, batalha por turnos, lojas.

---

## 3. Roguelike / roguelite (dungeon procedural)

**Núcleo:** níveis gerados, turnos, permadeath, itens com identificação,
progressão por run.

Arquitetura de referência (sistema de **atores e turnos**, como em roguelikes
pixel maduros):
- **Actor (base):** tudo que age no tempo é um Actor com um tempo/energia
  acumulado. Um *scheduler* processa o actor com menor tempo, ele age, gasta
  energia e é reagendado. Isso dá turnos de velocidades diferentes (um inimigo
  rápido age 2x).
- **Char → Hero / Mob:** personagens são Actors com HP, posição, IA. `Mob` tem
  estados (DORMINDO, VAGANDO, PERSEGUINDO, FUGINDO) e usa pathfinding no grid.
- **Buff:** efeitos temporários (veneno, fome, força) também são Actors anexados.
- **Item:** hierarquia (arma, armadura, poção, pergaminho, anel). Poções/
  pergaminhos começam **não identificados** (aparência aleatória por run) e são
  identificados ao usar/avaliar.
- **Level/Dungeon generation:** "painters" que escavam salas e corredores num
  grid (BSP, salas+corredores, autômato celular para cavernas), depois espalham
  mobs, itens, armadilhas e a escada de descida. Cada andar é um Level novo.
- **Save (bundle):** serialização de todo o estado do andar/herói para
  continuar a run.

Mecânicas-assinatura: campo de visão/fog of war, fome, identificação, andares
crescentes em dificuldade, morte permanente.

Geração procedural — técnicas:
- **Salas + corredores:** sorteia retângulos não sobrepostos, liga com túneis.
- **BSP:** subdivide o espaço recursivamente, cria uma sala por folha.
- **Autômato celular:** ruído + suavização → cavernas orgânicas.
- **Drunkard's walk:** caminhada aleatória escavando — cavernas conectadas.

---

## 4. Action-RPG / hack'n'slash (estilo Diablo)

**Núcleo:** combate em tempo real com muitos inimigos, loot, builds, atributos.

Sistemas:
- **Movimento e ataque** orientados ao mouse/direção; click-to-move ou WASD.
- **Stats e dano:** poder, precisão, esquiva, resistências; fórmulas de dano
  com tipos (físico, fogo...) e mitigação.
- **Loot por raridade** com afixos/modificadores aleatórios; tabelas de drop.
- **Habilidades/poderes** com cooldown, custo (mana), e árvores/builds.
- **Inimigos em grupo** com IA simples (perseguir + atacar) e elites/bosses.
- **Data-driven:** ARPGs maduros definem inimigos, itens, poderes e mapas em
  **arquivos de dados** (não código), permitindo modding e iteração rápida.
  Considere descrever conteúdo em JSON/INI e ter um motor genérico que o lê.

Engines especializadas em ARPG isométrico já trazem esse motor data-driven
pronto — escrever só os arquivos de conteúdo entrega um jogo completo.

---

## 5. Estratégia/tática por turnos (TBS / SRPG)

**Núcleo:** grid de batalha + unidades com stats + turnos + terreno + IA tática.

Sistemas:
- **Grid (quadrado/hex)** com custo de movimento por terreno e bônus defensivo
  por tipo de terreno.
- **Unidades:** classes, HP, ataque/defesa, alcance, custo de recrutamento,
  promoções/níveis.
- **Turnos por lado:** mover → atacar (cálculo com chance de acerto e dano,
  contra-ataque), depois IA do oponente.
- **Fog of war**, zonas de controle, vantagens de tipo (pedra-papel-tesoura
  entre classes), dia/noite afetando unidades.
- **Campanha:** sequência de cenários com objetivos, recrutamento persistente,
  história entre batalhas. Bom candidato a conteúdo data-driven (cenários e
  unidades em arquivos) e a suportar multiplayer hot-seat/online.

---

## 6. City builder / simulação (isométrico)

**Núcleo:** grid (isométrico) + colocação de construções + recursos +
simulação contínua.

Sistemas:
- **Grid isométrico** com tiles em losango; conversão tela↔grid e ordenação de
  desenho por profundidade (depth sort) para sobreposição correta.
- **Colocação:** ferramenta de construir/demolir, validação de terreno,
  estradas/conexões.
- **Recursos e economia:** produção/consumo, população, felicidade, finanças
  num *tick* de simulação periódico.
- **Camadas de mapa:** terreno, zonas, edifícios, redes (água/energia).
- **Modding-first:** definir edifícios e regras em arquivos de dados facilita
  expansão. Pense em ECS/data-driven para muitas entidades simuladas.

Atenção em isométrico: o **depth sorting** (o que desenha na frente) e o
**picking** (qual tile o mouse selecionou) são os dois problemas técnicos
centrais.

---

## Como escolher e escopar

- **Game jam / primeiro jogo:** plataforma de uma tela ou arena top-down. Pouca
  arte, loop fechado em dias.
- **Projeto médio:** roguelite (conteúdo proceduralmente reaproveitado) ou
  plataforma com várias fases.
- **Projeto grande:** RPG/metroidvania/tática — exigem muito conteúdo (arte,
  mapas, texto). Use ferramentas data-driven e considere engines de gênero.

Regra: o gênero define **quais** sistemas das outras referências você precisa
(IA, diálogo, inventário, geração procedural). Liste-os antes de codar.
