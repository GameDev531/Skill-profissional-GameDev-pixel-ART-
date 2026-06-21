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

## Checklist de mundo

- [ ] Tileset com tile size fixo e colisão marcada.
- [ ] Camadas separadas: bg / colisão / fg / entidades.
- [ ] Autotiling configurado (terrains embutidos ou bitmask 47).
- [ ] Object layer com spawns lidos pela engine.
- [ ] Greybox jogável validado antes da decoração final.
- [ ] Câmera com limites do nível e parallax de fundo.
