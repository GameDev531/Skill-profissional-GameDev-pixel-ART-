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

Serialize um **dicionário/struct de estado** (posição, stats, inventário,
flags de quest, progresso de fase) para JSON ou formato binário da engine.

```gdscript
# SaveSystem — autoload (Godot 4)
extends Node

const SAVE_PATH := "user://savegame.json"
const SAVE_VERSION := 1

func save_game(game_state: Dictionary) -> void:
    game_state["version"] = SAVE_VERSION
    var file := FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_string(JSON.stringify(game_state, "\t"))

func load_game() -> Dictionary:
    if not FileAccess.file_exists(SAVE_PATH):
        return {}
    var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
    var json := JSON.new()
    json.parse(file.get_as_text())
    var data: Dictionary = json.data
    if data.get("version", 0) < SAVE_VERSION:
        data = _migrate(data)
    return data

func _migrate(data: Dictionary) -> Dictionary:
    # Converta saves antigos para o formato atual
    return data

func delete_save() -> void:
    if FileAccess.file_exists(SAVE_PATH):
        DirAccess.remove_absolute(SAVE_PATH)
```

**Padrões-chave:**
- **Versão do save:** inclua um `version` no arquivo para migrar saves antigos.
- Salve em local apropriado por plataforma (`user://` no Godot).
- **Colete o estado com um grupo:** faça cada entidade saveable implementar
  `get_save_data() -> Dictionary` e itere com `get_tree().get_nodes_in_group("saveable")`.
- Teste o round-trip (salvar→carregar→comparar) num teste automatizado.

### Save/Load com Resources (abordagem nativa Godot 4)

Resources são serializáveis nativamente — o Godot salva/carrega toda a árvore de
propriedades automaticamente (inclusive Resources aninhados). Mais robusto que JSON
para dados complexos (Vector2, Color, Arrays tipados, sub-Resources).

```gdscript
# SaveGame — Resource que contém TODO o estado do save
class_name SaveGame
extends Resource

@export var player_stats: PlayerStats
@export var inventory: Inventory
@export var quest_data: Dictionary = {}
@export var world_state: Dictionary = {}
@export var play_time: float = 0.0
@export var save_date: String = ""
@export var current_scene: String = ""
@export var player_position: Vector2 = Vector2.ZERO

const SAVE_DIR := "user://saves/"
const SAVE_EXT := ".tres"

static func save_exists(slot: int) -> bool:
    return ResourceLoader.exists(_get_path(slot))

static func save_to_slot(data: SaveGame, slot: int) -> Error:
    data.save_date = Time.get_datetime_string_from_system()
    DirAccess.make_dir_recursive_absolute(SAVE_DIR)
    return ResourceSaver.save(data, _get_path(slot))

static func load_from_slot(slot: int) -> SaveGame:
    if not save_exists(slot):
        return null
    return load(_get_path(slot)) as SaveGame

static func delete_slot(slot: int) -> void:
    if save_exists(slot):
        DirAccess.remove_absolute(_get_path(slot))

static func _get_path(slot: int) -> String:
    return SAVE_DIR + "save_%d%s" % [slot, SAVE_EXT]
```

```gdscript
# Uso no GameManager (autoload):
func save(slot: int) -> void:
    var save_data := SaveGame.new()
    save_data.player_stats = player.stats.duplicate()
    save_data.inventory = player.inventory.duplicate()
    save_data.quest_data = QuestManager.serialize()
    save_data.current_scene = get_tree().current_scene.scene_file_path
    save_data.player_position = player.global_position
    save_data.play_time = _total_play_time
    # Coletar estado do mundo (baús abertos, NPCs derrotados, etc.)
    save_data.world_state = _collect_world_state()
    SaveGame.save_to_slot(save_data, slot)

func load_save(slot: int) -> void:
    var save_data := SaveGame.load_from_slot(slot)
    if not save_data: return
    await SceneTransition.change_scene(save_data.current_scene)
    player.stats = save_data.player_stats
    player.inventory = save_data.inventory
    player.global_position = save_data.player_position
    QuestManager.deserialize(save_data.quest_data)
    _apply_world_state(save_data.world_state)
    _total_play_time = save_data.play_time

func _collect_world_state() -> Dictionary:
    var state := {}
    for node in get_tree().get_nodes_in_group("saveable"):
        state[node.get_path()] = node.get_save_data()
    return state

func _apply_world_state(state: Dictionary) -> void:
    for node in get_tree().get_nodes_in_group("saveable"):
        var path := String(node.get_path())
        if state.has(path):
            node.load_save_data(state[path])
```

**Resource vs JSON — quando usar cada:**
| Resource (.tres) | JSON |
|---|---|
| Godot-only, serializa tudo nativo (Vector2, Color, sub-Resources) | Cross-engine, legível por humanos, editável manualmente |
| Seguro apenas se controlado (Resources podem conter scripts) | Sem risco de execução de código |
| Ideal para stats, inventário, configurações complexas | Ideal para dados web, leaderboards, configs simples |

**Segurança:** nunca carregue `.tres` de fontes externas sem validação — Resources
podem referenciar scripts. Para multiplayer/UGC, use JSON ou
`SafeResourceLoader` (verifica o conteúdo antes de carregar).

---

## 4b. Transição de cena (autoload)

```gdscript
# SceneTransition — autoload (CanvasLayer com ColorRect)
extends CanvasLayer

signal transition_started
signal scene_changed
signal transition_finished

@onready var color_rect: ColorRect = $ColorRect
@onready var anim_player: AnimationPlayer = $AnimationPlayer
var _is_transitioning := false

func change_scene(scene_path: String, duration: float = 0.4) -> void:
    if _is_transitioning: return
    _is_transitioning = true
    transition_started.emit()
    color_rect.visible = true
    anim_player.play("fade_out")
    await anim_player.animation_finished
    get_tree().change_scene_to_file(scene_path)
    scene_changed.emit()
    await get_tree().process_frame
    anim_player.play("fade_in")
    await anim_player.animation_finished
    color_rect.visible = false
    _is_transitioning = false
    transition_finished.emit()

func fade_only(callback: Callable, duration: float = 0.4) -> void:
    if _is_transitioning: return
    _is_transitioning = true
    color_rect.visible = true
    anim_player.play("fade_out")
    await anim_player.animation_finished
    callback.call()
    await get_tree().process_frame
    anim_player.play("fade_in")
    await anim_player.animation_finished
    color_rect.visible = false
    _is_transitioning = false
```

**Setup:** `CanvasLayer` (layer alto) com `ColorRect` fullscreen (preto,
alpha 0) + `AnimationPlayer` com tracks "fade_out" (alpha 0→1) e "fade_in"
(1→0). Registre como Autoload. Chame: `SceneTransition.change_scene("res://...")`.
O `fade_only` serve para mudanças dentro da mesma cena (ex.: trocar de sala).

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

## 6. Input remapping (rebind de controles em runtime)

Permite o jogador reconfigurar teclas/botões. Padrão: salvar em JSON, aplicar
no `InputMap` ao iniciar.

```gdscript
# InputRemapper — autoload
extends Node

const SAVE_PATH := "user://input_config.json"

var _default_events: Dictionary = {}

func _ready() -> void:
    _save_defaults()
    load_remaps()

func _save_defaults() -> void:
    for action in InputMap.get_actions():
        if action.begins_with("ui_"): continue
        _default_events[action] = InputMap.action_get_events(action).duplicate()

func remap_action(action: String, new_event: InputEvent) -> void:
    InputMap.action_erase_events(action)
    InputMap.action_add_event(action, new_event)
    save_remaps()

func reset_action(action: String) -> void:
    if _default_events.has(action):
        InputMap.action_erase_events(action)
        for event in _default_events[action]:
            InputMap.action_add_event(action, event)
    save_remaps()

func reset_all() -> void:
    for action in _default_events:
        reset_action(action)

func save_remaps() -> void:
    var data := {}
    for action in _default_events:
        var events := InputMap.action_get_events(action)
        var serialized := []
        for event in events:
            if event is InputEventKey:
                serialized.append({"type": "key", "keycode": event.physical_keycode})
            elif event is InputEventJoypadButton:
                serialized.append({"type": "joypad", "button": event.button_index})
        data[action] = serialized
    var file := FileAccess.open(SAVE_PATH, FileAccess.WRITE)
    file.store_string(JSON.stringify(data, "\t"))

func load_remaps() -> void:
    if not FileAccess.file_exists(SAVE_PATH): return
    var file := FileAccess.open(SAVE_PATH, FileAccess.READ)
    var json := JSON.new()
    if json.parse(file.get_as_text()) != OK: return
    var data: Dictionary = json.data
    for action in data:
        InputMap.action_erase_events(action)
        for entry in data[action]:
            var event: InputEvent
            match entry["type"]:
                "key":
                    event = InputEventKey.new()
                    event.physical_keycode = entry["keycode"]
                "joypad":
                    event = InputEventJoypadButton.new()
                    event.button_index = entry["button"]
            if event:
                InputMap.action_add_event(action, event)

func get_action_display_name(action: String) -> String:
    var events := InputMap.action_get_events(action)
    if events.is_empty(): return "[Não atribuído]"
    return events[0].as_text()
```

**UI de remap:** no menu de opções, ao clicar num slot de ação:
1. Mostra "Pressione uma tecla..."
2. Captura o próximo `InputEvent` com `_input(event)`
3. Chama `InputRemapper.remap_action(action_name, event)`
4. Atualiza o label do botão

---

## 7. Object Pool (reuso de instâncias)

Evita criar/destruir centenas de nós por frame (projéteis, partículas, inimigos).

```gdscript
class_name ObjectPool
extends Node

var _scene: PackedScene
var _pool: Array[Node] = []
var _active: Array[Node] = []
@export var initial_size: int = 20

func _init(scene: PackedScene = null, size: int = 20) -> void:
    _scene = scene
    initial_size = size

func _ready() -> void:
    for i in initial_size:
        var instance := _scene.instantiate()
        instance.set_process(false)
        instance.hide()
        add_child(instance)
        _pool.append(instance)

func get_instance() -> Node:
    var instance: Node
    if _pool.is_empty():
        instance = _scene.instantiate()
        add_child(instance)
    else:
        instance = _pool.pop_back()
    instance.set_process(true)
    instance.show()
    _active.append(instance)
    return instance

func release(instance: Node) -> void:
    if instance in _active:
        _active.erase(instance)
    instance.set_process(false)
    instance.hide()
    _pool.append(instance)

func release_all() -> void:
    for instance in _active.duplicate():
        release(instance)
```

**Uso típico (projéteis):**
```gdscript
@onready var bullet_pool := ObjectPool.new(preload("res://scenes/bullet.tscn"), 30)

func _ready() -> void:
    add_child(bullet_pool)

func shoot() -> void:
    var bullet := bullet_pool.get_instance() as Bullet
    bullet.global_position = muzzle.global_position
    bullet.direction = aim_direction
    bullet.setup(bullet_pool)  # bullet chama pool.release(self) ao sair da tela
```

No script do projétil:
```gdscript
var _pool: ObjectPool
func setup(pool: ObjectPool) -> void:
    _pool = pool
func _on_screen_exited() -> void:
    _pool.release(self)
```

---

## Checklist de produção

- [ ] Lógica crítica coberta por testes (dano, inventário, save, geração).
- [ ] UI desacoplada via sinais; pixel font + nine-patch; menus completos.
- [ ] Conteúdo em arquivos de dados, não hardcoded.
- [ ] `.gitignore` correto; assets grandes em LFS se preciso.
- [ ] Save/load versionado e testado.
- [ ] Build exportado e testado na plataforma-alvo, pixel-perfect.
