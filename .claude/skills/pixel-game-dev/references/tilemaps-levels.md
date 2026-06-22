# Tilemaps, Autotiling & Level Design

Como construir o mundo do jogo com tiles: editores, autotiling profissional,
formato de dados e import na engine.

---

## 1. Conceitos de tilemap

- **Tile:** célula quadrada de arte (8/16/32 px). O mundo é uma grade de tiles.
- **Tileset:** imagem com todos os tiles disponíveis, indexados.
- **Layer (camada):** um grid de índices de tile. Jogos usam várias:
  `background` (parallax/decoração, sem colisão), `ground/collision` (sólido),
  `foreground` (na frente do player), `entities` (spawns/objetos).
- **Colisão por tile:** marque quais tiles são sólidos (propriedade booleana
  `collides`) e gere os colliders a partir disso na engine.
- **Object layer:** posições de entidades (player start, inimigos, baús,
  gatilhos) — não são tiles, são pontos/retângulos com propriedades.

## 2. Editores de mapa (escolha)

| Editor | Forte em | Saída | Use quando |
|---|---|---|---|
| **Tiled** | Padrão da indústria; orthogonal, isométrico e hexagonal; object layers; propriedades custom; terrains/wang tiles | `.tmx`/`.tmj` (JSON) | Qualquer jogo tile-based; máximo de engines lê o formato |
| **LDtk** | Moderno; IntGrid + auto-layers (autotiling por regras), entidades com campos tipados, mundo multi-nível | `.ldtk` (JSON limpo) + "Super Simple Export" (PNG por camada + dados) | Quer autotiling poderoso e import simples |
| **Ogmo Editor 3** | Leve, simples, usado em fluxo indie; grid + entidades + decals | `.json` | Projeto enxuto, quer algo direto |

### Tiled — fluxo
1. Crie o tileset a partir do PNG (defina tile width/height).
2. Crie camadas de tile (`ground`, `bg`, `fg`) e pinte.
3. Marque colisão: adicione propriedade custom `collides=true` aos tiles
   sólidos, ou use uma camada de colisão dedicada.
4. Adicione uma **object layer** para spawns (player, inimigos, itens) com
   propriedades (`type=enemy`, `patrol=true`).
5. Exporte como JSON e importe na engine (Phaser tem `tilemapTiledJSON`; Godot
   importa via plugins; várias libs leem `.tmj`).

### LDtk — fluxo e conceitos
LDtk organiza o jogo em **World → Levels → Layers**:
- **IntGrid layer:** você pinta valores inteiros (1=parede, 2=água...). É a
  "verdade" lógica do nível; a arte é derivada dela por regras.
- **Auto-layer:** regras que olham a vizinhança de cada célula IntGrid e
  escolhem o tile certo automaticamente (autotiling baseado em regras/patterns).
- **Tiles layer:** pintura manual direta de tiles.
- **Entities layer:** instâncias com **campos tipados** (Int, Float, Bool,
  String, Enum, Point, cor, arrays, referências a outras entidades). Ex.: um
  `Enemy` com campos `hp:int`, `patrolPath:Point[]`.
- **Enums:** tipos enumerados reutilizáveis para tags/categorias.
- **World layout:** níveis posicionados num espaço 2D (GridVania, Free,
  Linear) — bom para metroidvania/mundo conectado.
- **Import:** ou leia o JSON `.ldtk` com um loader oficial (Haxe, Godot, Unity,
  GameMaker, etc.), ou use o **Super Simple Export** que gera um PNG por camada
  + um `data.json` mínimo — trivial de carregar em qualquer engine.

## 3. Autotiling profissional (blob 47-tiles)

Autotiling = escolher automaticamente o tile de borda/canto certo conforme os
vizinhos, para que paredes/terreno conectem sem você pintar cada caso à mão.

### Bitmask de 8 vizinhos → 47 tiles
Cada vizinho recebe um bit (potência de 2). Soma-se os bits dos vizinhos
"iguais" (mesmo terreno) para obter um índice:

```
Up-left   = 1     Up    = 2     Up-right   = 4
Left      = 8                   Right      = 16
Down-left = 32    Down  = 64    Down-right = 128
```

São 2^8 = **256** combinações possíveis, mas só **47** tiles únicos são
necessários por uma regra-chave:

> Um **vizinho diagonal só conta se os DOIS ortogonais adjacentes a ele também
> existem.** (Ex.: o canto up-right só importa se "up" E "right" existem.)

Isso elimina cantos "soltos" impossíveis e colapsa 256 → 47. Em código,
calcula-se o bitmask, aplica-se a regra de diagonais e usa-se uma **tabela de
lookup** (hash) que mapeia o valor para o índice do subtile no tileset.

### Variante 16-tiles (4-bit)
Olha só os 4 ortogonais (up/down/left/right) → 2^4 = **16** casos. Mais simples,
menos polido (cantos não tratados). Bom para começar; o 47 é o "profissional".

### Como montar o tileset
- Layout "blob" de 47 peças: um template (comumente 7×7 com algumas vazias)
  onde cada peça corresponde a um valor da tabela de lookup.
- Muitas engines têm autotiling embutido com terrains/peering bits (Godot
  TileSet "Terrains", Tiled "Wang/Terrain", LDtk auto-layer rules) — prefira o
  embutido; implemente o bitmask manual só em engines sem suporte (Raylib,
  Pygame, GameMaker via conversor de tiles RPG-Maker→autotile).

## 4. Level design — princípios

- **Bloque (greybox) antes de decorar.** Monte a fase só com tiles de colisão e
  valide o gameplay (pulos alcançáveis, ritmo) antes de gastar arte.
- **Ensine sem texto:** introduza cada mecânica num espaço seguro, depois
  combine, depois teste sob pressão (modelo "introduzir → desenvolver → desafiar").
- **Ritmo:** alterne tensão (combate/plataforma difícil) com alívio (exploração,
  checkpoint). Coloque checkpoints antes de seções duras.
- **Leitura visual:** use cor/luz para guiar o olhar até a saída ou objetivo.
  Foreground escuro para enquadrar, fundo de baixo contraste para profundidade.
- **Parallax:** camadas de fundo movendo em velocidades diferentes dão
  profundidade barata (mais lento = mais distante).
- **Limites de câmera:** trave a câmera nos bounds do nível para não mostrar o
  "vazio" fora do mapa.

## 5. Import na engine (resumo)

- **Godot:** TileMap/TileMapLayer + TileSet com física e terrains; objetos da
  object layer viram instâncias de cena (via plugin de import Tiled/LDtk).
- **Phaser:** `this.load.tilemapTiledJSON` + `map.createLayer`;
  `setCollisionByProperty({ collides: true })`; objetos via
  `map.getObjectLayer`.
- **Raylib/Pygame/Bevy:** leia o JSON, instancie tiles num array 2D, gere
  retângulos de colisão a partir dos tiles sólidos, faça spawn das entidades da
  object layer.

## 6. Geração procedural de dungeons

Três abordagens progressivas para criar layouts em runtime:

### 6a. BSP (Binary Space Partitioning) — salas + corredores

Divide recursivamente o espaço até atingir tamanho mínimo, cria uma sala em cada
folha, conecta salas irmãs por corredores.

```gdscript
class_name DungeonGenerator
extends Node

@export var grid_width: int = 50
@export var grid_height: int = 50
@export var min_room_size: int = 5
@export var min_partition_size: int = 10
@export var padding: int = 1

var grid: Array = []   # 1=parede, 0=chão
var rooms: Array[Rect2i] = []

class BSPNode:
    var x1: int; var y1: int; var x2: int; var y2: int
    var left: BSPNode; var right: BSPNode
    var room: Rect2i

func generate() -> void:
    grid.clear()
    rooms.clear()
    for y in grid_height:
        var row := []
        row.resize(grid_width)
        row.fill(1)
        grid.append(row)
    var root := BSPNode.new()
    root.x1 = 0; root.y1 = 0
    root.x2 = grid_width - 1; root.y2 = grid_height - 1
    _split(root)

func _split(node: BSPNode) -> void:
    var w := node.x2 - node.x1
    var h := node.y2 - node.y1
    if w <= min_partition_size or h <= min_partition_size:
        _create_room(node)
        return
    var left_n := BSPNode.new()
    var right_n := BSPNode.new()
    if w > h:
        var split := randi_range(node.x1 + min_partition_size, node.x2 - min_partition_size)
        left_n.x1 = node.x1; left_n.y1 = node.y1; left_n.x2 = split; left_n.y2 = node.y2
        right_n.x1 = split; right_n.y1 = node.y1; right_n.x2 = node.x2; right_n.y2 = node.y2
    else:
        var split := randi_range(node.y1 + min_partition_size, node.y2 - min_partition_size)
        left_n.x1 = node.x1; left_n.y1 = node.y1; left_n.x2 = node.x2; left_n.y2 = split
        right_n.x1 = node.x1; right_n.y1 = split; right_n.x2 = node.x2; right_n.y2 = node.y2
    node.left = left_n; node.right = right_n
    _split(left_n); _split(right_n)
    _connect(_get_room_center(left_n), _get_room_center(right_n))

func _create_room(node: BSPNode) -> void:
    var max_w := node.x2 - node.x1 - padding * 2
    var max_h := node.y2 - node.y1 - padding * 2
    var rw := randi_range(min_room_size, max(min_room_size, max_w))
    var rh := randi_range(min_room_size, max(min_room_size, max_h))
    var rx := randi_range(node.x1 + padding, max(node.x1 + padding, node.x2 - padding - rw))
    var ry := randi_range(node.y1 + padding, max(node.y1 + padding, node.y2 - padding - rh))
    node.room = Rect2i(rx, ry, rw, rh)
    rooms.append(node.room)
    for y in range(ry, ry + rh):
        for x in range(rx, rx + rw):
            grid[y][x] = 0

func _connect(a: Vector2i, b: Vector2i) -> void:
    for x in range(min(a.x, b.x), max(a.x, b.x) + 1):
        grid[a.y][x] = 0
    for y in range(min(a.y, b.y), max(a.y, b.y) + 1):
        grid[y][b.x] = 0

func _get_room_center(node: BSPNode) -> Vector2i:
    if node.room.size != Vector2i.ZERO:
        return node.room.position + node.room.size / 2
    return _get_room_center(node.left)

func get_random_floor_pos() -> Vector2i:
    var room: Rect2i = rooms[randi() % rooms.size()]
    return Vector2i(
        randi_range(room.position.x, room.end.x - 1),
        randi_range(room.position.y, room.end.y - 1))
```

**Aplicar à TileMapLayer:**
```gdscript
func apply_to_tilemap(tilemap: TileMapLayer, wall_id: int, floor_id: int) -> void:
    for y in grid_height:
        for x in grid_width:
            var atlas := Vector2i(floor_id, 0) if grid[y][x] == 0 else Vector2i(wall_id, 0)
            tilemap.set_cell(Vector2i(x, y), 0, atlas)
```

### 6b. Random Walk (drunk walk)

Mais orgânico — um "walker" caminha aleatoriamente cavando chão. Bom para caves.

```gdscript
func random_walk(steps: int, start: Vector2i) -> Array[Vector2i]:
    var path: Array[Vector2i] = [start]
    var directions := [Vector2i.UP, Vector2i.DOWN, Vector2i.LEFT, Vector2i.RIGHT]
    var pos := start
    for i in steps:
        pos += directions[randi() % 4]
        pos.x = clampi(pos.x, 1, grid_width - 2)
        pos.y = clampi(pos.y, 1, grid_height - 2)
        path.append(pos)
    return path

func generate_cave(walkers: int = 4, steps_per_walker: int = 200) -> void:
    for y in grid_height:
        for x in grid_width:
            grid[y][x] = 1
    var center := Vector2i(grid_width / 2, grid_height / 2)
    for w in walkers:
        for pos in random_walk(steps_per_walker, center):
            grid[pos.y][pos.x] = 0
```

### 6c. Cellular Automata (suavização de caves)

Pós-processo: gera ruído binário, depois aplica regras de vizinhança N vezes para
suavizar.

```gdscript
func cellular_automata(fill_chance: float = 0.45, iterations: int = 4) -> void:
    # Preenche aleatoriamente
    for y in grid_height:
        for x in grid_width:
            if x == 0 or x == grid_width - 1 or y == 0 or y == grid_height - 1:
                grid[y][x] = 1
            else:
                grid[y][x] = 1 if randf() < fill_chance else 0
    # Suaviza
    for i in iterations:
        var new_grid := grid.duplicate(true)
        for y in range(1, grid_height - 1):
            for x in range(1, grid_width - 1):
                var neighbors := _count_wall_neighbors(x, y)
                new_grid[y][x] = 1 if neighbors >= 5 else 0
        grid = new_grid

func _count_wall_neighbors(cx: int, cy: int) -> int:
    var count := 0
    for dy in range(-1, 2):
        for dx in range(-1, 2):
            if dx == 0 and dy == 0: continue
            if grid[cy + dy][cx + dx] == 1: count += 1
    return count
```

### Quando usar cada

| Algoritmo | Resultado | Bom para |
|---|---|---|
| BSP | Salas retangulares + corredores retos | Roguelikes clássicos, dungeons |
| Random Walk | Caves orgânicas, formas irregulares | Minas, cavernas, ruínas |
| Cellular Automata | Caves suaves com paredes arredondadas | Biomas naturais, cavernas |
| Combinação | BSP para layout → cellular para bordas | Dungeons com caves dentro |

---

## Checklist de mundo

- [ ] Tileset com tile size fixo e colisão marcada.
- [ ] Camadas separadas: bg / colisão / fg / entidades.
- [ ] Autotiling configurado (terrains embutidos ou bitmask 47).
- [ ] Object layer com spawns lidos pela engine.
- [ ] Greybox jogável validado antes da decoração final.
- [ ] Câmera com limites do nível e parallax de fundo.
- [ ] Geração procedural testada com seeds fixas para reprodutibilidade.
