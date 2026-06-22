# IA de Inimigos: Behavior Trees, State Machines & Pathfinding

Como dar comportamento a inimigos, bosses e NPCs. Três pilares: **decisão**
(behavior trees / máquinas de estado), **navegação** (pathfinding) e
**movimento** (steering).

---

## 1. Máquina de estados finita (FSM) — o básico que resolve 80%

Para a maioria dos inimigos pixel, uma FSM simples já basta: IDLE, PATROL,
CHASE, ATTACK, HURT, DEAD. Cada estado tem `enter/update/exit` e transições.

```gdscript
extends CharacterBody2D
enum State { IDLE, PATROL, CHASE, ATTACK, HURT, DEAD }
var state := State.PATROL
@onready var player := get_tree().get_first_node_in_group("player")

func _physics_process(delta):
    match state:
        State.PATROL:
            _patrol(delta)
            if _can_see_player(): state = State.CHASE
        State.CHASE:
            _move_towards(player.global_position, delta)
            if _in_attack_range(): state = State.ATTACK
            elif not _can_see_player(): state = State.PATROL
        State.ATTACK:
            _attack()
            if not _in_attack_range(): state = State.CHASE
    move_and_slide()
```

Quando a FSM "explode" (muitos estados/transições duplicadas) → suba para
**statecharts** ou **behavior trees**.

---

## 2. Statecharts (FSM hierárquica, sem state explosion)

Statecharts resolvem o problema de explosão de estados das FSMs tradicionais com
estados aninhados, paralelos e transições declarativas com guardas.

Tipos de estado:
- **Atomic:** estado-folha, sem filhos.
- **Compound:** contém sub-estados; exatamente UM filho ativo por vez (ex.:
  `Alive` contendo `Idle/Run/Jump`).
- **Parallel:** vários sub-estados ativos ao mesmo tempo (ex.: `Movement` e
  `Weapon` rodando em paralelo).
- **History:** ao reentrar num compound, volta ao último sub-estado ativo (bom
  para "retomar de onde parou" após um knockback).

Como funciona (padrão do StateChart para Godot):
- Você monta uma **árvore de nós de estado** sob um nó `StateChart`.
- Transições são **declarativas, com guardas** (condições) e podem ser
  **atrasadas no tempo** (ótimo para cooldowns).
- Seu código interage com **uma única classe** `StateChart`, com dois métodos
  principais: enviar evento e setar propriedade para guardas de expressão:
  ```gdscript
  @onready var state_chart: StateChart = $StateChart

  func _when_something_happened():
      state_chart.send_event("player_spotted")      # dispara transições

  func _on_hp_changed(new_hp: int):
      state_chart.set_expression_property("hp", new_hp) # alimenta guardas
  ```
- **Transições diretas (code-triggered):** invoque `.take()` num nó Transition:
  ```gdscript
  var transition: Transition = $StateChart/MyState/MyTransition
  transition.take()
  ```
- Cada nó de estado emite sinais que você conecta: `state_entered`,
  `state_exited`, `state_processing(delta)`, `state_physics_processing(delta)`,
  `state_input`, `event_received`. Você coloca a lógica do estado nesses sinais.

**Arquitetura do CompoundState** (estado composto com sub-estados):
```gdscript
class_name CompoundState extends StateChartState
signal child_state_entered()
signal child_state_exited()
@export_node_path("StateChartState") var initial_state: NodePath
var _active_state: StateChartState = null
```
- Na entrada, ativa o `initial_state` (ou restaura history state se aplicável).
- Ao receber transição, resolve se o alvo é filho direto, descendente ou externo
  e delega para cima/baixo conforme necessário.
- Na saída, salva o estado ativo nos HistoryStates antes de desativar.

**Guardas disponíveis:**
- `ExpressionGuard` — avalia expressão Godot usando propriedades do chart.
- `AllOfGuard` — AND lógico de sub-guardas.
- `AnyOfGuard` — OR lógico de sub-guardas.
- `NotGuard` — inverte a sub-guarda.
- `StateIsActiveGuard` — verdadeiro se um estado específico está ativo.

**Transições automáticas:** sem evento, "ficam em jogo" enquanto o estado está
ativo e disparam assim que a guarda se torna verdadeira. Reagem a
`set_expression_property` e mudanças de estado ativo automaticamente.

**Exemplo de platformer com statechart (separação de responsabilidades):**
```gdscript
func _on_jump_enabled_state_physics_processing(_delta):
    if Input.is_action_just_pressed("ui_accept"):
        velocity.y = JUMP_VELOCITY
        _state_chart.send_event("jump")
```
O princípio: **o statechart contém as regras de mudança de estado; o código
contém a lógica executada em cada estado.** Nunca verifique `if state == X` no
código — reaja aos sinais do statechart.

Use para player com muitos modos, inimigos com fases, bosses multi-fase e menus.

---

## 3. Behavior Trees (BT) — IA modular e escalável

BTs modelam IA como uma árvore percorrida por **ticks** periódicos. Cada nó
retorna um de três status:
- **SUCCESS** — concluiu com êxito.
- **FAILURE** — não conseguiu.
- **RUNNING** — ainda em andamento (continua nos próximos ticks).

Tipos de nó:
- **Composites** (controlam fluxo entre filhos):
  - *Sequence:* roda filhos em ordem; PARA no primeiro FAILURE; SUCCESS só se
    todos passarem (lógica "E"). Ex.: ver player → mirar → atirar.
  - *Selector* (fallback): tenta filhos em ordem; PARA no primeiro SUCCESS;
    FAILURE só se todos falharem (lógica "OU"). Ex.: atacar OU perseguir OU
    patrulhar.
- **Decorators** (envolvem 1 filho e modificam o resultado): inverter, repetir,
  limitar tentativas, *cooldown*, *time limit*, "sempre sucesso".
- **Leaves** (fazem o trabalho):
  - *Action:* executa (mover, atirar, tocar animação) e retorna status.
  - *Condition:* avalia estado (vejo o player? hp baixo?) → SUCCESS/FAILURE.

### Beehave (Godot 4) — implementação real

**Montagem:** adicione um `BeehaveTree` ao CharacterBody2D do inimigo. Abaixo,
monte a árvore com composites e leaves como nós filhos na cena.

**Estrutura de árvore típica:**
```
BeehaveTree
└── SelectorComposite (tenta atacar, senão patrulha)
    ├── SequenceComposite (atacar)
    │   ├── IsPlayerVisible (ConditionLeaf)
    │   ├── ChasePlayer (ActionLeaf)
    │   └── AttackPlayer (ActionLeaf)
    └── SequenceComposite (patrulhar)
        ├── MoveToPatrolPoint (ActionLeaf)
        └── WaitAtPatrolPoint (ActionLeaf)
```

**ConditionLeaf — verificar se o player está visível:**
```gdscript
class_name IsPlayerVisible extends ConditionLeaf
@export var detection_range := 200.0
@export var vision_cone_angle := 45.0

func tick(actor: Node, blackboard: Blackboard) -> int:
    var player = get_tree().get_first_node_in_group("player")
    if not player: return FAILURE
    var to_player = player.global_position - actor.global_position
    if to_player.length() > detection_range: return FAILURE
    var forward = Vector2.RIGHT.rotated(actor.rotation)
    if abs(forward.angle_to(to_player.normalized())) > deg_to_rad(vision_cone_angle):
        return FAILURE
    blackboard.set_value("player_position", player.global_position)
    return SUCCESS
```

**ActionLeaf — perseguir o player:**
```gdscript
class_name ChasePlayer extends ActionLeaf
@export var move_speed := 100.0
@export var attack_range := 30.0

func tick(actor: Node, blackboard: Blackboard) -> int:
    var target_pos = blackboard.get_value("player_position")
    if not target_pos: return FAILURE
    var dir = (target_pos - actor.global_position).normalized()
    actor.global_position += dir * move_speed * get_physics_process_delta_time()
    if actor.global_position.distance_to(target_pos) <= attack_range:
        return SUCCESS
    return RUNNING
```

**ActionLeaf — patrulhar com espera:**
```gdscript
class_name WaitAtPatrolPoint extends ActionLeaf
@export var wait_time := 2.0
var current := 0.0

func tick(actor: Node, blackboard: Blackboard) -> int:
    if blackboard.get_value("patrol_point_reached", false):
        blackboard.set_value("patrol_point_reached", false)
        current = 0.0
    current += get_physics_process_delta_time()
    return SUCCESS if current >= wait_time else RUNNING
```

**SelectorComposite (código real do motor Beehave):**
Itera filhos em ordem; no primeiro SUCCESS ou RUNNING, para e retorna esse
status. Se o filho anterior estava RUNNING e outro tem sucesso antes, o running
é interrompido (`interrupt()`). Se todos falharem, retorna FAILURE.

**Blackboard:** cada BeehaveTree tem um Blackboard (ou compartilhado). Use
`blackboard.set_value(key, value)` / `get_value(key)` para compartilhar dados
entre nós (posição do alvo, flags, cooldowns).

**Decorators disponíveis:** `InverterDecorator`, `RepeaterDecorator`,
`LimiterDecorator` (max N execuções), `CooldownDecorator` (tempo entre ticks),
`DelayDecorator` (espera antes de rodar), `TimeLimiterDecorator`,
`UntilFailDecorator`, `AlwaysSucceedDecorator`, `AlwaysFailDecorator`.

**Composites extras:** `SequenceStarComposite` (retoma do último running/failure),
`SelectorReactiveComposite`/`SequenceReactiveComposite` (reavalia condições
anteriores a cada tick), `RandomSelector`/`RandomSequence` (ordem aleatória),
`SimpleParallelComposite` (roda ação principal + background em paralelo).

**Lifecycle de cada nó:** `before_run` → N×`tick` → `after_run` (ou `interrupt`
se cortado pelo pai).

**BT + HSM juntos:** o padrão avançado usa um estado da máquina hierárquica que
**executa um behavior tree** (um `BTState`). Assim você combina estados de alto
nível (Patrulha/Combate/Fuga) com lógica detalhada em BT dentro de cada um.

Quando usar BT vs FSM/statechart:
- Inimigo simples (3–5 estados) → FSM ou statechart.
- IA reativa/complexa, bosses, NPCs com muitas prioridades → behavior tree.

---

## 4. Pathfinding (navegação no grid)

### A* em grid (conceito)
A* acha o caminho mais curto combinando custo real (g) + heurística (h). Em
grid, a heurística é a distância estimada até o alvo:
- **Manhattan** (sem diagonal), **Chebyshev/Octile** (com diagonal),
  **Euclidean**.

### Godot — AStarGrid2D (exemplo real com TileMapLayer)
Integração completa A* + TileMap, incluindo consulta de walkability e desenho:
```gdscript
extends TileMapLayer

const CELL_SIZE = Vector2i(64, 64)

var _astar := AStarGrid2D.new()
var _start_point := Vector2i()
var _end_point := Vector2i()
var _path := PackedVector2Array()

func _ready() -> void:
    _astar.region = Rect2i(0, 0, 18, 10)
    _astar.cell_size = CELL_SIZE
    _astar.offset = CELL_SIZE * 0.5
    _astar.default_compute_heuristic = AStarGrid2D.HEURISTIC_MANHATTAN
    _astar.default_estimate_heuristic = AStarGrid2D.HEURISTIC_MANHATTAN
    _astar.diagonal_mode = AStarGrid2D.DIAGONAL_MODE_NEVER
    _astar.update()
    # Marca tiles usadas como sólidas (paredes/obstáculos)
    for pos in get_used_cells():
        _astar.set_point_solid(pos)
        # Filtre por atlas coords ou custom data:
        # if get_cell_tile_data(pos).get_custom_data("type") == "obstacle":

func is_point_walkable(local_position: Vector2) -> bool:
    var map_position := local_to_map(local_position)
    if _astar.is_in_boundsv(map_position):
        return not _astar.is_point_solid(map_position)
    return false

func find_path(local_start: Vector2i, local_end: Vector2i) -> PackedVector2Array:
    _start_point = local_to_map(local_start)
    _end_point = local_to_map(local_end)
    _path = _astar.get_point_path(_start_point, _end_point)
    queue_redraw()
    return _path.duplicate()

func _draw() -> void:
    if _path.is_empty(): return
    var last := _path[0]
    for i in range(1, _path.size()):
        draw_line(last, _path[i], Color.WHITE * Color(1,1,1,0.5), 3.0, true)
        draw_circle(_path[i], 6.0, Color.WHITE * Color(1,1,1,0.5))
        last = _path[i]
```

**Configurações comuns de AStarGrid2D:**
- `diagonal_mode`: `DIAGONAL_MODE_NEVER` (4-way), `DIAGONAL_MODE_ONLY_IF_NO_OBSTACLES`, `DIAGONAL_MODE_ALWAYS`
- `jumping_enabled = true`: permite pular sobre obstáculos isolados
- `region` ou `size`: define a área do grid (use `tilemap.get_used_rect()`)
- Heurísticas: `HEURISTIC_MANHATTAN` (4-dir), `HEURISTIC_OCTILE` (8-dir), `HEURISTIC_EUCLIDEAN`

(Há também `AStar2D`/`AStar3D` para grafos arbitrários, e nós utilitários de
A* em grid 2D que encapsulam isso com obstáculos dinâmicos.)

### Web/JS — biblioteca de pathfinding
Para jogos web, uma lib dedicada oferece vários algoritmos:
```js
var grid = new PF.Grid(matrix);              // 0=livre, 1=bloqueado
grid.setWalkableAt(x, y, false);
var finder = new PF.AStarFinder({
  allowDiagonal: true,
  dontCrossCorners: true,
  heuristic: PF.Heuristic.octile
});
var path = finder.findPath(sx, sy, tx, ty, grid);   // [[x,y],...]
path = PF.Util.smoothenPath(grid, path);            // suaviza/compacta
```
Finders disponíveis: **AStarFinder**, **DijkstraFinder** (sem heurística, custo
uniforme), **BestFirstFinder** (guloso, rápido mas não-ótimo), **BreadthFirst**
**Finder**, **JumpPointFinder** (A* otimizado para grids uniformes, muito mais
rápido), e variantes bidirecionais (Bi*). **Importante:** o grid é modificado
durante a busca — **clone** (`grid.clone()`) antes de reusar.

Escolha: caminho mais curto garantido → A*/Dijkstra/JPS. Rápido e "bom o
bastante" → BestFirst. Grids grandes e uniformes → JumpPoint.

### NavMesh
Para mundos não-grid, use navigation regions/navmesh da engine (Godot
`NavigationRegion2D` + `NavigationAgent2D`) — o agente pathfinda e evita
obstáculos automaticamente.

---

## 5. Steering behaviors (movimento orgânico)

Steering produz movimento suave somando forças de direção (em vez de seguir o
path "robótico"):
- **Seek / Flee:** ir até / fugir de um ponto.
- **Arrive:** chegar desacelerando (sem ultrapassar).
- **Pursue / Evade:** perseguir/fugir prevendo a posição futura do alvo.
- **Wander:** vagar aleatório suave (patrulha natural).
- **Separation / Cohesion / Alignment:** comportamento de **flock/boids**
  (grupos de inimigos que não se empilham).
- **Obstacle/collision avoidance**, **path following**, **formation motion**
  (grupos em formação).

Combine steering + pathfinding: A* dá os waypoints, *path following* + *arrive*
suaviza o trajeto, *separation* evita amontoamento. Frameworks de IA de jogo
trazem steering, pathfinding (incl. A* indexado/hierárquico), behavior trees,
FSM (incl. `StackStateMachine`) e **troca de mensagens** entre agentes
(message dispatcher/telegraph com delay) prontos.

---

## Guia de escolha rápido

| Necessidade | Ferramenta |
|---|---|
| Inimigo com poucos estados | FSM manual |
| Player/inimigo com modos aninhados, cooldowns | Statechart |
| IA reativa rica, boss, prioridades | Behavior Tree (+ blackboard) |
| Caminho em grid de tiles | A* de grid (embutido) |
| Pathfinding em jogo web | lib JS (A*/JPS) |
| Mundo livre, evitar obstáculos | NavMesh/NavigationAgent |
| Movimento suave, grupos | Steering / boids |
