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
  $StateChart.send_event("player_spotted")      # dispara transições
  $StateChart.set_expression_property("hp", hp) # alimenta guardas
  ```
- Cada nó de estado emite sinais que você conecta: `state_entered`,
  `state_exited`, `state_processing(delta)`, `state_physics_processing(delta)`,
  `state_input`, `event_received`. Você coloca a lógica do estado nesses sinais.

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

Como integrar (padrão das libs de BT para Godot):
- Monte a árvore como nós na cena e anexe a qualquer node; um `BTPlayer`/runner
  faz o tick a cada frame.
- **Blackboard:** memória compartilhada entre os nós (ex.: `target`,
  `last_seen_pos`). Use *scopes* para evitar conflito de nomes entre agentes.
- **Tasks customizadas:** estenda as classes-base (`BTAction`, `BTCondition`,
  `BTDecorator`, `BTComposite`) e implemente o `tick()`:

```gdscript
extends BTAction          # task customizada
func _tick(delta) -> Status:
    var target = blackboard.get_var("target")
    if target == null: return FAILURE
    agent.move_towards(target.global_position, delta)
    return RUNNING if not agent.reached(target) else SUCCESS
```

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

### Godot — AStarGrid2D
Godot tem A* de grid embutido:
```gdscript
var astar := AStarGrid2D.new()
astar.region = Rect2i(0, 0, map_w, map_h)
astar.cell_size = Vector2(16, 16)
astar.diagonal_mode = AStarGrid2D.DIAGONAL_MODE_NEVER
astar.update()
# marque sólidos a partir do TileMap:
for cell in tilemap.get_used_cells_by_id(...):
    astar.set_point_solid(cell, true)
var path := astar.get_point_path(start_cell, target_cell)  # Array[Vector2]
```
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
