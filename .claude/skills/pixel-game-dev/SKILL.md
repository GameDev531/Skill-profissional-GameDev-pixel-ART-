---
name: pixel-game-dev
description: >-
  Guia completo e profissional para criar jogos pixel art 2D do zero ao
  jogável. Use quando o usuário quiser desenvolver um jogo pixel/retrô, escolher
  engine (Godot, Phaser, Pyxel, Raylib, Bevy, Solarus, Flare), implementar
  mecânicas (plataforma, RPG top-down, roguelike, ARPG, tática por turnos),
  criar pixel art e animação de sprites, montar mapas com tiles e autotiling,
  programar IA de inimigos (behavior trees, A*, máquinas de estado), diálogo e
  narrativa ramificada, shaders/efeitos/"game feel", áudio retrô, multiplayer ou
  testes. Cobre o fluxo end-to-end, do protótipo à exportação.
---

# Pixel Game Dev — Criação Profissional de Jogos Pixel Art 2D

Esta skill ensina a projetar e programar jogos pixel art completos. O conteúdo
é destilado de dezenas de engines, editores de arte, editores de mapa, sistemas
de IA, narrativa, shaders e ferramentas de teste de referência. Use-a como
manual de decisão e implementação: ela diz **o que escolher**, **por que**,
**como estruturar** e **como implementar** cada sistema, com código pronto.

## Princípio central

Um jogo pixel é a soma de 6 sistemas. Construa nesta ordem — cada um destrava o
próximo e evita retrabalho:

1. **Loop + movimento** — janela, game loop, input, um personagem que se move.
2. **Mundo** — tilemap, colisão, câmera, fases (level design).
3. **Arte** — sprites, animação, paleta, spritesheets, "game feel".
4. **Gameplay** — mecânicas do gênero (combate, itens, inimigos, progressão).
5. **IA & sistemas** — pathfinding, behavior trees, diálogo, UI, save/load.
6. **Polish** — shaders, partículas, áudio, screen shake, menus, testes.

> Regra de ouro: tenha algo **jogável em minutos**, não perfeito em meses.
> Comece com um retângulo que pula numa fase de teste antes de desenhar arte.

## Como usar esta skill (roteiro de decisão)

1. **Defina escopo e gênero.** Plataforma, RPG top-down, roguelike, ARPG,
   metroidvania, tática por turnos ou city builder? → `references/genres-mechanics.md`
2. **Escolha a engine** pela linguagem do usuário e plataforma-alvo. Use a
   tabela de decisão abaixo → detalhes e padrões de código em
   `references/engines-frameworks.md`
3. **Defina a resolução base e a paleta** ANTES de desenhar qualquer pixel →
   `references/pixel-art.md`
4. **Monte o pipeline de mapas** (Tiled ou LDtk) e autotiling →
   `references/tilemaps-levels.md`
5. **Implemente os sistemas** conforme o gênero pede: IA, narrativa, multiplayer,
   shaders/áudio, testes → arquivos de referência específicos.
6. **Faça polish e exporte:** game feel, áudio, menus, save/load e build para a
   plataforma-alvo → `references/shaders-audio-juice.md` e
   `references/testing-tooling.md`.

## Tabela de decisão de engine

| Quero… | Engine recomendada | Linguagem | Por quê |
|---|---|---|---|
| Tudo-em-um, 2D first, grátis, GUI completa | **Godot 4** | GDScript | Maior ecossistema 2D pixel; IA, diálogo e shaders integram nativamente |
| Jogo web/HTML5 ou mobile rápido | **Phaser 3** | JavaScript/TS | Roda no navegador, deploy trivial, muitos exemplos |
| Estética retrô estrita (paleta fixa, 256x256) | **Pyxel** | Python | Impõe limites retrô; ótimo para fan/game-jam |
| Controle baixo-nível, C/C++, aprender fundamentos | **Raylib** | C/C++ | API simples, sem "mágica"; ótimo para entender o loop |
| Arquitetura data-driven / ECS, performance | **Bevy** | Rust | ECS moderno; bom para sistemas complexos |
| Action-RPG estilo Zelda/Diablo pronto | **Solarus** / **Flare** | Lua / dados | Engines especializadas, já trazem o gênero pronto |
| Prototipar mecânica em Python | **Pygame** | Python | Mínimo atrito para testar ideias |

Não conhece a linguagem do usuário? Pergunte. Na dúvida geral, **Godot 4** é o
default mais seguro para pixel art 2D.

## Não-negociáveis de pixel art (resumo)

- **Escolha UMA resolução base** (ex.: 320×180, 16:9 retrô) e **um tamanho de
  tile** (8, 16 ou 32 px). Nunca misture escalas de pixel na mesma cena.
- **Pixel perfect**: câmera e renderização em inteiros; desligue filtro
  bilinear (use *nearest*); escale a janela por múltiplos inteiros (×2, ×3, ×4).
- **Paleta limitada** (16–32 cores) dá coesão. Defina antes de produzir.
- **Animação**: pense em poses-chave (idle, walk, jump, hit, death). 4–8 frames
  já vendem o movimento. → `references/pixel-art.md`

## Arquivos de referência

Carregue sob demanda conforme a tarefa:

- `references/engines-frameworks.md` — escolha de engine, estrutura de projeto e
  **padrões de código prontos** (controller de plataforma, top-down, câmera) em
  Godot, Phaser, Pyxel, Raylib, Bevy, Pygame, Solarus, Flare.
- `references/pixel-art.md` — fundamentos de pixel art, resolução, paleta,
  animação, spritesheets, ferramentas (Pixelorama/LibreSprite/Piskel/LPC).
- `references/tilemaps-levels.md` — Tiled, LDtk, Ogmo, autotiling 47-tiles,
  fluxo de level design e import na engine.
- `references/genres-mechanics.md` — anatomia e mecânicas-núcleo de cada gênero
  (plataforma, RPG, roguelike, ARPG, tática, city builder).
- `references/ai-pathfinding.md` — behavior trees (Beehave/LimboAI), statecharts,
  máquinas de estado, A*/Dijkstra/steering.
- `references/narrative-dialogue.md` — Yarn Spinner, Ink, Dialogic; diálogo
  ramificado, escolhas e visual novel.
- `references/shaders-audio-juice.md` — shaders 2D, partículas, screen shake,
  "game feel" e áudio retrô (jsfxr).
- `references/multiplayer.md` — tempo real (Colyseus) e backend social/match
  (Nakama).
- `references/testing-tooling.md` — testes (gdUnit4), UI de jogo (HUD/menus/
  nine-patch), versionamento, save/load, organização de projeto e exportação.

## Checklist de "jogo pronto pra mostrar"

- [ ] Movimento responsivo com *coyote time* e *jump buffer* (plataforma) ou
      movimento em grid/8-direções suave (top-down).
- [ ] Câmera pixel-perfect que segue o player com limites de fase.
- [ ] Pelo menos 1 fase de teste e 1 fase "real" feitas em Tiled/LDtk.
- [ ] Sprite animado para idle/walk/ação principal.
- [ ] 1 inimigo com IA (patrulha + perseguição) e colisão de dano.
- [ ] Feedback de impacto: flash de hit, partícula, screen shake, som.
- [ ] Menu inicial, pause e tela de game over.
- [ ] Save/load do progresso básico.
- [ ] Exportação para a plataforma-alvo testada.
