# Testes, UI, Organização de Projeto & Exportação

Práticas que mantêm o jogo sustentável: testes automatizados, UI de jogo,
estrutura de projeto, versionamento e build/exportação.

---

## 1. Testes automatizados (ex.: gdUnit4 no Godot)

Mesmo em jogo, teste a **lógica pura** (dano, inventário, geração procedural,
save/load) e os **fluxos de cena** com um runner.

Estrutura de teste:
```gdscript
class_name PlayerStatsTest
extends GdUnitTestSuite

func before():        # uma vez antes da suíte
    pass
func before_test():   # antes de cada teste
    pass

func test_damage_reduces_hp():
    var p := Player.new()
    p.hp = 100
    p.take_damage(30)
    assert_int(p.hp).is_equal(70)

func test_inventory_stacks():
    var inv := Inventory.new()
    inv.add("potion", 2)
    inv.add("potion", 3)
    assert_int(inv.count("potion")).is_equal(5)
```

API de asserts (sintaxe fluente, encadeável):
- `assert_int()`, `assert_str()`, `assert_bool()`, `assert_float()`,
  `assert_array()`, `assert_dict()`, `assert_object()`, `assert_vector()`,
  `assert_signal()`.
- Encadeie: `assert_str(msg).has_length(26).starts_with("This is")`.

**Scene Runner** — simula input/frames para testar cenas reais:
- Simula clique/teclado/toque/ações de input.
- Avança N frames (`simulate_frames`) e espera por sinais/retornos.
- Bom para testar "apertar pulo faz o player subir", "inimigo morre ao tomar
  N de dano".

Lifecycle: `before`/`after` (suíte), `before_test`/`after_test` (cada teste).
Rode pela UI da engine (menu de contexto no FileSystem/ScriptEditor) ou por
**linha de comando** para CI/CD.

O que vale testar num jogo:
- Fórmulas de combate/dano e regras de stats.
- Inventário/equipamento (adicionar, empilhar, remover, equipar).
- Save/load (round-trip: salvar → carregar → estado idêntico).
- Geração procedural (níveis sempre conectados, sem sala isolada).
- Máquinas de estado (transições válidas).

---

## 2. UI de jogo

UI de jogo (HUD, menus, inventário, diálogo) precisa ser **escalável em pixel**
e responsiva a resolução.

Princípios:
- **Use o sistema de UI da engine** (Godot `Control`/`Container`/anchors; Phaser
  containers) com âncoras e *layout containers* para adaptar a tamanhos de tela.
- **Pixel UI:** desenhe a UI na mesma resolução base e escale por inteiro;
  fontes bitmap (pixel font) mantêm a estética. Evite fontes vetoriais
  suavizadas que destoam.
- **9-slice (nine-patch):** caixas/painéis que esticam sem distorcer as bordas —
  ideal para janelas de diálogo e botões de tamanho variável.
- **Separe UI de lógica:** a UI escuta sinais/eventos do jogo (ex.: player emite
  `health_changed(hp)`, a barra de vida atualiza). Não acesse a UI direto da
  lógica de gameplay.
- **Estados de UI:** menu inicial, opções, pause, game over, HUD in-game,
  inventário — modele como cenas/telas trocáveis.
- Ferramentas dedicadas de **layout de UI de jogo** existem para montar telas
  com componentes e layout responsivo, exportando para a engine — úteis quando a
  UI fica complexa (RPG com muitas janelas).

Componentes comuns: barra de vida/mana, minimapa, hotbar/slots, tooltips,
janela de diálogo (com nine-patch + pixel font + efeito de digitação), menus
navegáveis por teclado/gamepad.

---

## 3. Organização de projeto e versionamento

Estrutura recomendada (adapte à engine):
```
project/
  scenes|src/      # entidades, níveis, telas
  scripts/         # lógica reutilizável (sem depender de cena)
  assets/
    sprites/  tilesets/  audio/  fonts/  shaders/
  data/            # JSON/recursos: itens, inimigos, diálogos, níveis
  tests/
  export/          # builds (ignorado no git)
```
- **Data-driven:** mantenha conteúdo (itens, inimigos, balanceamento, diálogos)
  em arquivos de dados, não hardcoded. Acelera iteração e permite modding.
- **Singletons/autoloads** para estado global: `GameState`, `AudioManager`,
  `SaveSystem`, `SceneTransition`.
- **Sinais/eventos** para desacoplar sistemas (gameplay ↔ UI ↔ áudio).

Git para jogos:
- **`.gitignore`** builds, caches de import e binários temporários (ex.:
  `.godot/`, `export/`, `node_modules/`, `.import/`).
- **Git LFS** para assets binários grandes (PNGs grandes, áudio) se o repo
  crescer.
- Commits pequenos e descritivos; branch por feature.
- Versione os arquivos de projeto da engine e os dados; não versione builds.

---

## 4. Save/load

- Serialize um **dicionário/struct de estado** (posição, stats, inventário,
  flags de quest, progresso de fase) para JSON ou formato binário da engine.
- **Versão do save:** inclua um `version` no arquivo para migrar saves antigos.
- Salve em local apropriado por plataforma (`user://` no Godot).
- Teste o round-trip (salvar→carregar→comparar) num teste automatizado.

---

## 5. Exportação / build

- **Defina a plataforma-alvo cedo** (desktop Win/Mac/Linux, web/HTML5, mobile,
  consoles) — afeta engine, controles e UI.
- **Web/HTML5:** ótimo para alcance e demos (Phaser nativo; Godot/Pyxel exportam
  pra web). Cuide do tamanho do build e do carregamento de assets.
- **Desktop:** empacote por SO; teste fullscreen + janela e a escala inteira em
  várias resoluções.
- **Mobile:** repense controles (toque/virtual pad) e densidade de tela.
- **Pixel-perfect na exportação:** confirme filtro nearest e escala inteira na
  build final, em diferentes resoluções de tela.
- Automatize builds por linha de comando (CI) quando possível e teste o
  executável final, não só o editor.

---

## Checklist de produção

- [ ] Lógica crítica coberta por testes (dano, inventário, save, geração).
- [ ] UI desacoplada via sinais; pixel font + nine-patch; menus completos.
- [ ] Conteúdo em arquivos de dados, não hardcoded.
- [ ] `.gitignore` correto; assets grandes em LFS se preciso.
- [ ] Save/load versionado e testado.
- [ ] Build exportado e testado na plataforma-alvo, pixel-perfect.
