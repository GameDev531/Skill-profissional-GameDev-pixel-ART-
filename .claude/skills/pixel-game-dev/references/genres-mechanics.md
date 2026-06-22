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

### Controlador de plataforma profissional (implementação completa)

Sistema com pulo baseado em física real (altura → gravidade calculada), coyote
time, jump buffer, pulo variável (soltar = menor), double jump e falling gravity
multiplier.

```gdscript
extends CharacterBody2D
class_name PlatformerController2D

signal jumped(is_ground_jump: bool)
signal hit_ground()

@export var input_left: String = "move_left"
@export var input_right: String = "move_right"
@export var input_jump: String = "jump"

@export var max_jump_height: float = 150.0:
    set(value):
        _max_jump_height = value
        _recalculate_physics()
@export var min_jump_height: float = 60.0
@export var double_jump_height: float = 100.0
@export var jump_duration: float = 0.3:
    set(value):
        _jump_duration = value
        _recalculate_physics()
@export var falling_gravity_multiplier: float = 1.5
@export var max_jump_amount: int = 1
@export var max_acceleration: float = 10000.0
@export var friction: float = 20.0
@export var can_hold_jump: bool = false
@export var coyote_time: float = 0.1
@export var jump_buffer: float = 0.1

var _max_jump_height: float
var _jump_duration: float
var default_gravity: float
var jump_velocity: float
var double_jump_velocity: float
var release_gravity_multiplier: float

var jumps_left: int
var holding_jump := false
var _was_on_ground: bool

enum JumpType { NONE, GROUND, AIR }
var current_jump_type: JumpType = JumpType.NONE

@onready var coyote_timer := Timer.new()
@onready var jump_buffer_timer := Timer.new()

func _init():
    _recalculate_physics()

func _ready():
    if coyote_time > 0:
        add_child(coyote_timer)
        coyote_timer.wait_time = coyote_time
        coyote_timer.one_shot = true
    if jump_buffer > 0:
        add_child(jump_buffer_timer)
        jump_buffer_timer.wait_time = jump_buffer
        jump_buffer_timer.one_shot = true

func _physics_process(delta: float) -> void:
    if not coyote_timer.is_stopped() or current_jump_type == JumpType.NONE:
        jumps_left = max_jump_amount
    if is_on_floor() and current_jump_type == JumpType.NONE:
        coyote_timer.start()

    if not _was_on_ground and is_on_floor():
        current_jump_type = JumpType.NONE
        if not jump_buffer_timer.is_stopped() and not can_hold_jump:
            jump()
        hit_ground.emit()

    if Input.is_action_pressed(input_jump) and can_hold_jump:
        if _can_ground_jump():
            jump()

    var gravity := _apply_gravity_multipliers(default_gravity)
    var acc := Vector2()
    if Input.is_action_pressed(input_left): acc.x = -max_acceleration
    if Input.is_action_pressed(input_right): acc.x = max_acceleration
    acc.y = gravity

    velocity.x *= 1.0 / (1.0 + delta * friction)
    velocity += acc * delta
    _was_on_ground = is_on_floor()
    move_and_slide()

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed(input_jump):
        holding_jump = true
        jump_buffer_timer.start()
        if (not can_hold_jump and _can_ground_jump()) or _can_double_jump():
            jump()
    elif event.is_action_released(input_jump):
        holding_jump = false

func jump() -> void:
    if _can_double_jump():
        velocity.y = -double_jump_velocity
        current_jump_type = JumpType.AIR
        if jumps_left == max_jump_amount:
            jumps_left -= 1
        jumps_left -= 1
        jumped.emit(false)
    else:
        velocity.y = -jump_velocity
        current_jump_type = JumpType.GROUND
        jumps_left -= 1
        coyote_timer.stop()
        jumped.emit(true)

func _can_ground_jump() -> bool:
    return (jumps_left > 0 and current_jump_type == JumpType.NONE) \
        or not coyote_timer.is_stopped()

func _can_double_jump() -> bool:
    if jumps_left <= 1 and jumps_left == max_jump_amount:
        return false
    return jumps_left > 0 and not is_on_floor() and coyote_timer.is_stopped()

func _apply_gravity_multipliers(gravity: float) -> float:
    if velocity.y > 0:
        gravity *= falling_gravity_multiplier
    elif velocity.y < 0 and not holding_jump:
        if current_jump_type != JumpType.AIR:
            gravity *= release_gravity_multiplier
    return gravity

func _recalculate_physics() -> void:
    default_gravity = (2.0 * _max_jump_height) / pow(_jump_duration, 2)
    jump_velocity = (2.0 * _max_jump_height) / _jump_duration
    double_jump_velocity = sqrt(abs(2.0 * default_gravity * double_jump_height))
    release_gravity_multiplier = (pow(jump_velocity, 2) / (2.0 * min_jump_height)) / default_gravity
```

**Fórmulas de pulo baseadas em game design (não em valores mágicos):**
- Gravidade = `2h / t²` onde h=altura máx, t=tempo até o pico
- Vel. pulo = `2h / t`
- Isso permite tunar "quero pular 3 tiles de altura em 0.3s" diretamente.

**Coyote time:** ~0.08–0.12s permite pular logo após cair de plataforma.
**Jump buffer:** ~0.1–0.15s registra input apertado antes de aterrissar.
**Pulo variável:** ao soltar jump, multiplica gravidade para encurtar o arco.

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

### Sistema de combate turn-based — arquitetura real (Godot 4)

Padrão de sistema de combate profissional para RPGs 2D:

**BattlerStats (Resource):** dados numéricos com modificadores empilháveis.
```gdscript
class_name BattlerStats extends Resource
const MODIFIABLE_STATS = ["max_health","max_energy","attack","defense","speed","hit_chance","evasion"]
signal health_depleted
signal health_changed

@export var base_max_health := 100
@export var base_attack := 10
@export var base_defense := 10
@export var base_speed := 70
@export var base_hit_chance := 100
@export var base_evasion := 0

var health := max_health:
    set(value):
        health = clampi(value, 0, max_health)
        health_changed.emit()
        if health == 0: health_depleted.emit()

var _modifiers := {}    # { "attack": { id: value } }
var _multipliers := {}  # multiplicadores somados (min 0.0)

func add_modifier(stat_name: String, value: int) -> int:
    # retorna id único para remover depois (ex.: ao desequipar)
    ...

func _recalculate_and_update(prop_name: String) -> void:
    var value := get("base_" + prop_name) as float
    var stat_multiplier := 1.0
    for m in _multipliers[prop_name].values(): stat_multiplier += m
    stat_multiplier = max(stat_multiplier, 0.0)
    value *= stat_multiplier
    for mod in _modifiers[prop_name].values(): value += mod
    set(prop_name, roundf(max(value, 0.0)))
```
Conceito-chave: base_X + modifiers (aditivos de equipamento/buff) + multipliers
(percentuais). IDs permitem remover modificadores individualmente (útil para
equipar/desequipar e expirar buffs).

**Battler (Node2D):** entidade combatente com estados e sinais.
```gdscript
class_name Battler extends Node2D
signal turn_finished
signal health_depleted
signal hit_received(value: int)

@export var stats: BattlerStats
@export var actions: Array[BattlerAction]
@export var ai_scene: PackedScene  # IA para inimigos
var is_active := true
var cached_action: BattlerAction = null

func act() -> void:
    stats.energy -= cached_action.energy_cost
    await cached_action.execute()
    cached_action = null
    turn_finished.emit.call_deferred()

func take_hit(hit: BattlerHit) -> void:
    if hit.is_successful():
        hit_received.emit(hit.damage)
        stats.health -= hit.damage
    else: hit_missed.emit()
```

**BattlerAction (Resource):** ações abstratas com escopo de alvo.
```gdscript
class_name BattlerAction extends Resource
enum TargetScope { SELF, SINGLE, ALL }
@export var target_scope := TargetScope.SINGLE
@export var targets_enemies := true
@export var energy_cost := 0

func can_execute() -> bool:
    return source.stats.energy >= energy_cost and not get_possible_targets().is_empty()

func execute() -> void:  # override em cada ação concreta
    pass
```

**Fluxo de batalha:** turnos por velocidade (`Battler.sort` por `stats.speed`),
UI mostra ações → player escolhe ação+alvo → `cached_action` setado → `act()`
chamado → animação → `turn_finished` → próximo battler. Inimigos usam `CombatAI`
para selecionar ação automaticamente.

### Inimigo RPG top-down — padrão real (SimpleRPG)

```gdscript
extends KinematicBody2D
signal death
var health := 100
var attack_damage := 10
var speed := 25
var direction := Vector2.ZERO
var player: Node

func _process(delta):
    health = min(health + 1 * delta, 100)  # regen
    var relative := player.position - position
    if relative.length() <= 16:
        direction = Vector2.ZERO  # perto: parar e virar
    elif relative.length() <= 100:
        direction = relative.normalized()  # range: perseguir
    elif randf() < 0.05:
        direction = Vector2.ZERO  # longe: vagar/ficar parado
    elif randf() < 0.1:
        direction = Vector2.DOWN.rotated(randf() * TAU)

func _physics_process(delta):
    var collision = move_and_collide(direction * speed * delta)
    if collision and collision.collider.name != "Player":
        direction = direction.rotated(randf_range(PI/4, PI/2))

func hit(damage):
    health -= damage
    if health <= 0:
        emit_signal("death")
        # drop item com 80% de chance
        if randf() <= 0.8:
            var potion = potion_scene.instance()
            get_tree().root.call_deferred("add_child", potion)
            potion.position = position
        player.add_xp(25)

func to_dictionary(): return {"position": [position.x, position.y], "health": health}
func from_dictionary(data):
    position = Vector2(data.position[0], data.position[1])
    health = data.health
```
Padrões mostrados: IA simples (range-based), regen, drops por probabilidade,
serialização para save/load via dicionário.

### Hitbox / Hurtbox — combate action em tempo real (Godot 4)

Padrão fundamental para qualquer jogo com ataque melee/ranged:

```gdscript
# HitBox — área de ataque (espada, projétil, armadilha)
class_name HitBox
extends Area2D

signal hit

@export var damage := 1
var id := -1  # ID única por ataque (evita multi-hit no mesmo swing)

func _ready() -> void:
    monitoring = false  # ativada só durante o ataque

func refresh_id() -> void:
    id = randi()  # chame antes de cada novo ataque
```

```gdscript
# HurtBox — área que recebe dano (corpo do inimigo/player)
class_name HurtBox
extends Area2D

signal hurt(damage: int)

@export var health: Node  # referência ao componente de vida
var last_hit_id := -1

func _ready() -> void:
    monitorable = false
    area_entered.connect(_on_area_entered)

func _on_area_entered(area: Area2D) -> void:
    if not area is HitBox: return
    var hitbox := area as HitBox
    if hitbox.id == last_hit_id: return  # ignora mesmo ataque
    last_hit_id = hitbox.id
    if health:
        health.take_damage(hitbox.damage)
    hitbox.hit.emit()
    hurt.emit(hitbox.damage)
```

**Collision layers (exemplo):**
- Layer 1: Player hurtbox
- Layer 2: Enemy hurtbox
- Layer 3: Player hitbox (mask → layer 2)
- Layer 4: Enemy hitbox (mask → layer 1)

**Padrão de uso:**
1. Espada/ataque: `HitBox` filho do player, `monitoring = false` por padrão.
2. No frame de ataque (via `AnimationPlayer`), ative `hitbox.monitoring = true`
   e chame `hitbox.refresh_id()`.
3. No fim do ataque, `hitbox.monitoring = false`.
4. Inimigos/player têm `HurtBox` sempre ativa; ao colidir com `HitBox`, aplica
   dano + i-frames (desabilita hurtbox por ~0.5s).

**I-frames (invincibilidade pós-dano):**
```gdscript
func take_damage(amount: int) -> void:
    if _invulnerable: return
    hp -= amount
    _invulnerable = true
    # Flash branco via shader (ver shaders-audio-juice.md)
    $HurtBox/CollisionShape2D.set_deferred("disabled", true)
    await get_tree().create_timer(0.5).timeout
    $HurtBox/CollisionShape2D.disabled = false
    _invulnerable = false
```

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

Geração procedural — técnicas e implementação real (Godot 4):

**1. Salas + corredores (Brogue-style, do godot-roguelike-example):**
```gdscript
# Colocar salas aleatórias sem sobreposição num grid booleano
var grid: Array[Array]  # true = ocupado
func _generate_dungeon_rooms(width, height, params) -> Array[Room]:
    var rooms: Array[Room] = []
    var attempts := 0
    while attempts < 500 and rooms.size() < 20:
        var room_w := rng.randi_range(min_size, max_size)
        var room_h := rng.randi_range(min_size, max_size)
        var room_x := rng.randi_range(border, width - room_w - border)
        var room_y := rng.randi_range(border, height - room_h - border)
        # Verificar se sobrepõe (com 1-cell buffer)
        var can_place := true
        for x in range(room_x - 1, room_x + room_w + 1):
            for y in range(room_y - 1, room_y + room_h + 1):
                if grid[x][y]: can_place = false; break
        if can_place:
            for x in range(room_x, room_x + room_w):
                for y in range(room_y, room_y + room_h): grid[x][y] = true
            rooms.append(Room.new(room_x, room_y, room_w, room_h))
        attempts += 1
    return rooms
```

**2. Conexão via MST (Minimum Spanning Tree + Kruskal):**
```gdscript
# Garante que todas as salas estejam conectadas sem ciclos
func _connect_all_rooms(map, rooms):
    var connections: Array[RoomConnection] = []
    for i in range(rooms.size()):
        for j in range(i + 1, rooms.size()):
            connections.append(RoomConnection.new(i, j, center_distance(rooms[i], rooms[j])))
    connections.sort_custom(func(a,b): return a.distance < b.distance)
    var ds := DisjointSet.new(rooms.size())
    for c in connections:
        if ds.find(c.room1_id) != ds.find(c.room2_id):
            ds.union(c.room1_id, c.room2_id)
            _connect_rooms(map, rooms[c.room1_id], rooms[c.room2_id])  # corredor L-shaped
```

**3. BSP (Binary Space Partition):**
```gdscript
func _generate_bsp_rooms(x, y, w, h, depth, min_room, min_split) -> Array[Room]:
    if depth <= 0:
        # Folha: criar sala com padding dentro do retângulo
        return [Room.new(x+2, y+2, w-4, h-4)]
    if randf() < 0.4 and h > min_split:  # split horizontal
        var split := y + h/2
        return _generate_bsp_rooms(x,y,w,split-y, depth-1, ...) + \
               _generate_bsp_rooms(x,split,w,h-(split-y), depth-1, ...)
    elif w > min_split:  # split vertical
        var split := x + w/2
        return _generate_bsp_rooms(x,y,split-x,h, depth-1, ...) + \
               _generate_bsp_rooms(split,y,w-(split-x),h, depth-1, ...)
    else:
        return [Room.new(x+1, y+1, w-2, h-2)]
```

**4. Populamento (itens, monstros, obstáculos por tipo de sala):**
- Cada sala recebe um `Room.Type` (EMPTY, LIBRARY, CRYPT, ALTAR...) com
  `ObstacleConfig` específico (estantes, caixões, altares).
- Monstros e itens distribuídos por `_place_monsters(count)` /
  `_place_items(count)` com tries limitados (10x) por posição.
- Itens empilháveis com quantidade; containers com itens dentro.
- Escadas up/down conectam andares (`destination_level`).
- Portas entre corredor↔sala (80% fechada, 20% aberta).

**5. Autômato celular** (para cavernas orgânicas):
```
1. Gerar ruído (45-55% chance de ser parede por célula)
2. Repetir 4-5x: se uma célula tem ≥5 vizinhos-parede nos 8 adjacentes → vira parede; senão → chão
3. Resultado: cavernas orgânicas conectadas
```

**6. Drunkard's walk:** agente que anda aleatoriamente e escava. Quando % do
mapa escavado atingir o alvo (30–40%), para. Garante conexão natural.

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

### Arquitetura de referência (builder 2D em Godot)

**Hierarquia de classes:**
```gdscript
# Entity — base de qualquer item colocado no mundo
class_name Entity
extends Node2D

export var deconstruct_filter: String
export var pickup_count := 1

func toggle_outline(enabled: bool) -> void:
    for sprite in _sprites:
        if sprite.material:
            sprite.material.set_shader_param("line_thickness",
                3.0 if enabled else 0.0)

func _setup(_blueprint: BlueprintEntity) -> void:
    pass  # override em subclasses (FurnaceEntity, WireEntity, etc.)
```

```gdscript
# BlueprintEntity — representação de item no inventário e durante colocação
class_name BlueprintEntity
extends Node2D

export var stack_size := 1
export var placeable := true
export (String, MULTILINE) var description := ""
var stack_count := 1

func make_inventory() -> void:
    scale = Vector2(gui_scale, gui_scale)
    modulate = Color.white

func make_world() -> void:
    scale = Vector2.ONE
    position = Vector2.ZERO

func rotate_blueprint() -> void:
    # Rotaciona direções de energia (LEFT→UP→RIGHT→DOWN)
    pass
```

**EntityTracker — registro de posições no grid:**
```gdscript
class_name EntityTracker
extends Reference

var entities := {}

func place_entity(entity, cellv: Vector2) -> void:
    if entities.has(cellv): return
    entities[cellv] = entity
    Events.emit_signal("entity_placed", entity, cellv)

func remove_entity(cellv: Vector2) -> void:
    if entities.has(cellv):
        var entity = entities[cellv]
        entities.erase(cellv)
        Events.emit_signal("entity_removed", entity, cellv)
        entity.queue_free()

func is_cell_occupied(cellv: Vector2) -> bool:
    return entities.has(cellv)

func get_entity_at(cellv: Vector2) -> Node2D:
    return entities.get(cellv)
```

**EntityPlacer — TileMap que gerencia construção/desconstrução:**
```gdscript
class_name EntityPlacer
extends TileMap

const MAXIMUM_WORK_DISTANCE := 275.0
var _tracker: EntityTracker

func _unhandled_input(event: InputEvent) -> void:
    var cellv := world_to_map(get_global_mouse_position())
    var cell_is_occupied := _tracker.is_cell_occupied(cellv)
    var is_close := global_mouse_position.distance_to(
        _player.global_position) < MAXIMUM_WORK_DISTANCE
    var is_on_ground := _ground.get_cellv(cellv) == 0

    if event.is_action_pressed("left_click"):
        if has_placeable_blueprint and not cell_is_occupied and is_close and is_on_ground:
            _place_entity(cellv)
    elif event.is_action_pressed("right_click"):
        if cell_is_occupied and is_close:
            _deconstruct(cellv)

func _place_entity(cellv: Vector2) -> void:
    var new_entity := Library.entities[blueprint_name].instance()
    add_child(new_entity)
    new_entity.global_position = map_to_world(cellv) + POSITION_OFFSET
    new_entity._setup(blueprint)
    _tracker.place_entity(new_entity, cellv)
    blueprint.stack_count -= 1
```

**PowerSystem — rede de energia com pathfinding de grafos:**
```gdscript
class_name PowerSystem
extends Reference

var power_sources := {}   # cellv → PowerSource
var power_receivers := {} # cellv → PowerReceiver
var power_movers := {}    # cellv → Entity (fios)
var paths := []           # caminhos source→receivers

func _on_systems_ticked(delta: float) -> void:
    for path in paths:
        var source: PowerSource = power_sources[path[0]]
        var available_power := source.get_effective_power()
        for cell in path.slice(1, path.size() - 1):
            if not power_receivers.has(cell): continue
            var receiver: PowerReceiver = power_receivers[cell]
            var required := receiver.get_effective_power()
            var delivered := min(available_power, required)
            receiver.emit_signal("received_power", delivered, delta)
            available_power -= delivered
            if available_power == 0: break
        source.emit_signal("power_updated", available_power, delta)

func _retrace_paths() -> void:
    paths.clear()
    for source_cell in power_sources.keys():
        paths.push_back(_trace_path_from(source_cell, [source_cell]))
```

**WorkComponent — crafting com progresso:**
```gdscript
class_name WorkComponent
extends Node

signal work_accomplished(amount)
signal work_done(output)
signal work_enabled_changed(enabled)

var current_output: BlueprintEntity
var available_work := 0.0
var work_speed := 0.0
var is_enabled := false

func setup_work(inputs: Dictionary, recipe_map: Dictionary) -> bool:
    for output in recipe_map.keys():
        var can_craft := true
        for input in inputs.keys():
            if inputs[input] < recipe_map[output].inputs.get(input, INF):
                can_craft = false; break
        if can_craft:
            current_output = Library.blueprints[output].instance()
            current_output.stack_count = recipe_map[output].amount
            available_work = recipe_map[output].time
            return true
    return false

func work(delta: float) -> void:
    if is_enabled and available_work > 0.0:
        available_work -= delta * work_speed
        emit_signal("work_accomplished", delta * work_speed)
        if available_work <= 0.0:
            emit_signal("work_done", current_output)
```

**Padrão geral:** entidades são colocadas no grid via `EntityPlacer`, registradas
no `EntityTracker` (dict cell→entidade), e sistemas (`PowerSystem`, `WorkSystem`)
reagem a sinais `entity_placed`/`entity_removed` para atualizar suas listas.
Blueprints definem a representação de inventário; Entities definem o comportamento
no mundo. Separação **inventário ↔ mundo** é a chave arquitetural.

---

## 7. Sistema de inventário (Resource-based, Godot 4)

Padrão para qualquer gênero que precise de itens (RPG, survival, builder):

```gdscript
# ItemData — definição de um tipo de item (Resource, data-driven)
class_name ItemData
extends Resource

enum ItemType { CONSUMABLE, EQUIPMENT, MATERIAL, KEY }

@export var name: String
@export var icon: Texture2D
@export var item_type: ItemType
@export var stackable: bool = true
@export var max_stack: int = 99
@export var description: String

func can_stack_with(other: ItemData) -> bool:
    return stackable and resource_path == other.resource_path

func apply_effects(stats) -> void:
    pass  # override em subclasses (HealItem, BuffItem, etc.)
```

```gdscript
# Inventory — container de slots com stacking e sinais
class_name Inventory
extends Resource

signal item_added(item: ItemData, slot: int)
signal item_removed(item: ItemData, slot: int)
signal item_used(item: ItemData, slot: int)
signal inventory_changed

class InventorySlot:
    var item: ItemData
    var count: int
    func _init(p_item: ItemData = null, p_count: int = 0) -> void:
        item = p_item; count = p_count

@export var size: int = 20
var slots: Array[InventorySlot] = []

func _init() -> void:
    for i in range(size):
        slots.append(InventorySlot.new())

func add_item(item: ItemData, amount: int = 1) -> bool:
    # 1) Tenta stackar com slots existentes
    if item.stackable:
        for i in range(slots.size()):
            var slot = slots[i]
            if slot.item and slot.item.can_stack_with(item) and slot.count < item.max_stack:
                var space = item.max_stack - slot.count
                var add_amount = min(amount, space)
                slot.count += add_amount
                amount -= add_amount
                item_added.emit(item, i)
                if amount <= 0:
                    inventory_changed.emit()
                    return true
    # 2) Slots vazios para o restante
    for i in range(slots.size()):
        if slots[i].item == null:
            slots[i].item = item
            slots[i].count = min(amount, item.max_stack)
            amount -= slots[i].count
            item_added.emit(item, i)
            if amount <= 0:
                inventory_changed.emit()
                return true
    inventory_changed.emit()
    return amount <= 0

func remove_item(slot_index: int, amount: int = 1) -> bool:
    var slot = slots[slot_index]
    if slot.item == null or slot.count < amount: return false
    slot.count -= amount
    item_removed.emit(slot.item, slot_index)
    if slot.count <= 0:
        slot.item = null; slot.count = 0
    inventory_changed.emit()
    return true

func use_item(slot_index: int, stats) -> bool:
    var slot = slots[slot_index]
    if slot.item == null: return false
    if slot.item.item_type == ItemData.ItemType.CONSUMABLE:
        slot.item.apply_effects(stats)
        remove_item(slot_index)
        item_used.emit(slot.item, slot_index)
        return true
    return false
```

**Padrões de inventário:**
- Itens como **Resource** (`ItemData`) permitem definir tudo no Inspector e
  serializar/salvar facilmente.
- `Inventory` é Resource → pode ser salvo em disco com `ResourceSaver`.
- A UI observa `inventory_changed` e redesenha slots.
- Equipment slots: um segundo `Inventory` (size=6, não-stackable) com validação
  de `ItemType.EQUIPMENT`.
- Drag & drop: `_get_drag_data` / `_can_drop_data` / `_drop_data` nos painéis.

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
