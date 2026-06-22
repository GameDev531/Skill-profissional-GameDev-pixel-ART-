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

### Sistema de combate turn-based — arquitetura real (Godot 4)

Padrão extraído de sistemas profissionais (GDQuest Open RPG):

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
