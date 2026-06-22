# Engines & Frameworks — Escolha, Estrutura e Padrões de Código

Guia prático por engine: quando usar, como organizar o projeto e **código
inicial pronto** (loop, input, movimento, câmera pixel-perfect).

---

## 1. Godot 4 (GDScript) — default recomendado para pixel 2D

**Use quando:** quer um motor 2D-first, grátis, com editor visual, e quer
integrar IA/diálogo/shaders desta lista sem fricção.

**Configuração pixel-perfect (Project Settings):**
- `Rendering > Textures > Default Texture Filter = Nearest`
- `Display > Window > Stretch > Mode = viewport`, `Aspect = keep`
- Defina resolução base baixa (ex.: 320×180) e deixe o `viewport` escalar.
- Em cada `Camera2D`, ative *pixel snap* / use `subpixel` desligado.

**Estrutura de projeto recomendada:**
```
res://
  scenes/      # .tscn por entidade/level (Player, Enemy, Level01)
  scripts/     # .gd reutilizáveis (state machine, utils)
  art/         # sprites, tilesets
  audio/
  ui/
  autoload/    # singletons (GameState, AudioManager, SaveSystem)
```
Use **Autoloads (singletons)** para estado global e o sistema de **Signals**
para desacoplar (player emite `health_changed`, UI escuta).

**Controller de plataforma 2D (com coyote time + jump buffer):**
```gdscript
extends CharacterBody2D

const SPEED := 120.0
const JUMP_VELOCITY := -300.0
const ACCEL := 1200.0
const FRICTION := 1400.0
var gravity: float = ProjectSettings.get_setting("physics/2d/default_gravity")

@export var coyote_time := 0.1
@export var jump_buffer := 0.1
var _coyote := 0.0
var _buffer := 0.0

func _physics_process(delta: float) -> void:
    # gravidade
    if not is_on_floor():
        velocity.y += gravity * delta
        _coyote -= delta
    else:
        _coyote = coyote_time

    # buffer de pulo
    if Input.is_action_just_pressed("jump"):
        _buffer = jump_buffer
    _buffer -= delta
    if _buffer > 0.0 and _coyote > 0.0:
        velocity.y = JUMP_VELOCITY
        _buffer = 0.0
        _coyote = 0.0
    # pulo variável (soltar cedo = pulo mais baixo)
    if Input.is_action_just_released("jump") and velocity.y < 0:
        velocity.y *= 0.5

    # movimento horizontal com aceleração/atrito
    var dir := Input.get_axis("move_left", "move_right")
    if dir != 0.0:
        velocity.x = move_toward(velocity.x, dir * SPEED, ACCEL * delta)
        $Sprite2D.flip_h = dir < 0.0
    else:
        velocity.x = move_toward(velocity.x, 0.0, FRICTION * delta)

    move_and_slide()
```

**Top-down 8 direções (RPG):**
```gdscript
extends CharacterBody2D
const SPEED := 90.0
func _physics_process(_delta):
    var input := Input.get_vector("left","right","up","down")
    velocity = input * SPEED
    move_and_slide()
    if input != Vector2.ZERO:
        $AnimationTree.set("parameters/blend_position", input)
```

**Câmera que segue com limites:** use `Camera2D` filho do player, defina
`limit_left/right/top/bottom` pelos bounds do TileMap, ative `position_smoothing`.

> Padrões a dominar em Godot 4: cena raiz por entidade (`CharacterBody2D` +
> `Sprite2D`/`AnimatedSprite2D` + `CollisionShape2D`), estrutura PRO com boss/
> inventário/loja/UI desacoplados por sinais, RPG turn-based com fila de turnos,
> roguelike com geração procedural por andar, e template top-down mínimo (player
> + TileMap + câmera) como ponto de partida copiável.

---

## 2. Phaser 3 (JavaScript/TypeScript) — jogo web/mobile

**Use quando:** quer rodar no navegador, deploy fácil (itch.io/web), mobile.

**Pixel-perfect:** no config `pixelArt: true` e `roundPixels: true`; escale por
`scale.zoom`.

**Esqueleto de cena com física Arcade:**
```js
const config = {
  type: Phaser.AUTO,
  width: 320, height: 180,
  pixelArt: true,
  scale: { mode: Phaser.Scale.FIT, zoom: 3, autoCenter: Phaser.Scale.CENTER_BOTH },
  physics: { default: 'arcade', arcade: { gravity: { y: 600 }, debug: false } },
  scene: { preload, create, update }
};
new Phaser.Game(config);

function preload() {
  this.load.spritesheet('hero', 'hero.png', { frameWidth: 16, frameHeight: 16 });
  this.load.tilemapTiledJSON('map', 'level01.json');
  this.load.image('tiles', 'tileset.png');
}
function create() {
  const map = this.make.tilemap({ key: 'map' });
  const tiles = map.addTilesetImage('tileset', 'tiles');
  const ground = map.createLayer('ground', tiles);
  ground.setCollisionByProperty({ collides: true });

  this.player = this.physics.add.sprite(32, 32, 'hero');
  this.player.setCollideWorldBounds(true);
  this.physics.add.collider(this.player, ground);

  this.anims.create({ key: 'walk',
    frames: this.anims.generateFrameNumbers('hero', { start: 0, end: 3 }),
    frameRate: 8, repeat: -1 });

  this.cursors = this.input.keyboard.createCursorKeys();
  this.cameras.main.startFollow(this.player).setBounds(0,0,map.widthInPixels,map.heightInPixels);
}
function update() {
  const p = this.player, c = this.cursors, speed = 100;
  p.setVelocityX(0);
  if (c.left.isDown) { p.setVelocityX(-speed); p.flipX = true; p.anims.play('walk', true); }
  else if (c.right.isDown) { p.setVelocityX(speed); p.flipX = false; p.anims.play('walk', true); }
  else p.anims.stop();
  if (c.up.isDown && p.body.blocked.down) p.setVelocityY(-300);
}
```

**Padrão de jogo completo (runner/dungeon):**
```js
class Game extends Phaser.Scene {
  preload() {
    this.registry.set("score", "0");
    this.load.audio("jump", "assets/jump.mp3");
    this.load.audio("coin", "assets/coin.mp3");
    this.load.audio("death", "assets/death.mp3");
    this.load.spritesheet("hero", "hero.png", { frameWidth: 16, frameHeight: 16 });
  }
  create() {
    this.width = this.sys.game.config.width;
    this.height = this.sys.game.config.height;
    // Player com física
    this.player = this.physics.add.sprite(32, this.height - 32, "hero");
    this.player.setCollideWorldBounds(true);
    // Input: espaço e clique
    this.input.on("pointerdown", () => this.jump());
    this.input.keyboard.on("keydown-SPACE", () => this.jump());
    // Colisões com callbacks
    this.physics.add.collider(this.player, this.obstacles, () => this.hitObstacle());
    this.physics.add.overlap(this.player, this.coins, (p, c) => this.hitCoin(c));
    // Câmera
    this.cameras.main.startFollow(this.player);
  }
  jump() {
    if (this.player.body.blocked.down) {
      this.player.setVelocityY(-300);
      this.sound.play("jump");
    }
  }
  hitCoin(coin) {
    this.sound.play("coin");
    coin.destroy();
    this.registry.set("score", parseInt(this.registry.get("score")) + 10);
  }
  hitObstacle() {
    this.sound.play("death");
    this.cameras.main.shake(200, 0.01);
    this.scene.start("GameOver");
  }
}
```

> Padrões a dominar em Phaser: organizar o jogo em **Scenes** (Boot, Preload,
> Menu, Game, UI) trocáveis; física Arcade (gravidade, colliders, overlaps);
> `tilemapTiledJSON` para mapas; spritesheets + `anims.create`; `registry` para
> estado global entre cenas; `cameras.main.shake/fade/startFollow` para juice.
> Para multiplayer em tempo real, integre com salas autoritativas (ver
> `multiplayer.md`).

---

## 3. Pyxel (Python) — estética retrô estrita

**Use quando:** quer limites retrô reais (16 cores, 256×256, 4 canais de som) e
prototipar rápido em Python. Ótimo para game jams e fan-games.

```python
import pyxel

class App:
    def __init__(self):
        pyxel.init(160, 120, title="Pixel Game", fps=60)
        pyxel.load("assets.pyxres")  # sprites/tiles/som no editor embutido
        self.x, self.y = 80, 60
        pyxel.run(self.update, self.draw)

    def update(self):
        if pyxel.btn(pyxel.KEY_LEFT):  self.x -= 1
        if pyxel.btn(pyxel.KEY_RIGHT): self.x += 1
        if pyxel.btnp(pyxel.KEY_SPACE): pyxel.play(0, 0)  # som

    def draw(self):
        pyxel.cls(0)
        pyxel.blt(self.x, self.y, 0, 0, 0, 16, 16, 0)  # sprite do banco 0

App()
```
Pyxel traz **editor de sprite, tilemap, som e música embutidos** (`pyxel edit`).

> Padrões a dominar em Pyxel: classe `App` com `update()`/`draw()`, banco de
> assets `.pyxres` (editado em `pyxel edit`), `blt` para sprites e `bltm` para
> tilemaps, e os 4 canais de som com `play`/`playm`.

---

## 4. Raylib (C/C++) — fundamentos baixo-nível

**Use quando:** quer entender o loop sem mágica, performance nativa, ou ensinar
fundamentos. API procedural e direta.

```c
#include "raylib.h"
int main(void) {
    const int W = 320, H = 180;
    InitWindow(W*3, H*3, "Pixel Game");
    SetTargetFPS(60);
    RenderTexture2D target = LoadRenderTexture(W, H);  // render em baixa res
    Texture2D hero = LoadTexture("hero.png");
    Vector2 pos = { 100, 80 };

    while (!WindowShouldClose()) {
        if (IsKeyDown(KEY_RIGHT)) pos.x += 1.5f;
        if (IsKeyDown(KEY_LEFT))  pos.x -= 1.5f;

        BeginTextureMode(target);          // desenha no buffer pequeno
            ClearBackground(BLACK);
            DrawTexture(hero, (int)pos.x, (int)pos.y, WHITE);
        EndTextureMode();

        BeginDrawing();                    // escala inteiro p/ tela
            ClearBackground(BLACK);
            DrawTexturePro(target.texture,
              (Rectangle){0,0,(float)W,-(float)H},
              (Rectangle){0,0,(float)W*3,(float)H*3},
              (Vector2){0,0}, 0, WHITE);
        EndDrawing();
    }
    CloseWindow();
    return 0;
}
```
A técnica do `RenderTexture` (renderizar em baixa resolução e escalar por
inteiro) é o padrão pixel-perfect em qualquer engine baixo-nível.

> Padrões a dominar em Raylib: o ciclo `BeginDrawing/EndDrawing`, render num
> `RenderTexture2D` de baixa resolução escalado por inteiro, `Camera2D` com
> target/offset/zoom para seguir o player com suavização, e checkpoints +
> geração procedural de fases num jogo de plataforma.

---

## 5. Bevy (Rust) — arquitetura ECS/data-driven

**Use quando:** quer ECS moderno, sistemas paralelos, projetos maiores onde
arquitetura importa. Curva mais íngreme.

Conceitos-chave: **Entity** (id), **Component** (dado puro), **System** (função
que opera sobre queries de componentes), **Resource** (estado global).

```rust
use bevy::prelude::*;

#[derive(Component)]
struct Player;

fn setup(mut commands: Commands, asset_server: Res<AssetServer>) {
    commands.spawn(Camera2dBundle::default());
    commands.spawn((
        SpriteBundle { texture: asset_server.load("hero.png"), ..default() },
        Player,
    ));
}

fn move_player(keys: Res<ButtonInput<KeyCode>>,
               mut q: Query<&mut Transform, With<Player>>) {
    let mut t = q.single_mut();
    if keys.pressed(KeyCode::ArrowRight) { t.translation.x += 2.0; }
    if keys.pressed(KeyCode::ArrowLeft)  { t.translation.x -= 2.0; }
}

fn main() {
    App::new()
        .add_plugins(DefaultPlugins.set(ImagePlugin::default_nearest())) // pixel
        .add_systems(Startup, setup)
        .add_systems(Update, move_player)
        .run();
}
```

> Padrões a dominar em Bevy: pensar em **Components** (dados), **Systems**
> (lógica por query) e **Resources** (estado global); `Startup` vs `Update`
> schedules; `ImagePlugin::default_nearest()` para pixel; sprite sheets via
> `TextureAtlas`.

---

## 6. Pygame (Python) — protótipos mínimos

**Use quando:** quer testar uma mecânica com o mínimo de cerimônia.

```python
import pygame
pygame.init()
SCALE = 3
screen = pygame.display.set_mode((320*SCALE, 180*SCALE))
canvas = pygame.Surface((320, 180))  # render em baixa res
clock = pygame.time.Clock()
x, y = 160, 90
running = True
while running:
    for e in pygame.event.get():
        if e.type == pygame.QUIT: running = False
    keys = pygame.key.get_pressed()
    if keys[pygame.K_LEFT]:  x -= 2
    if keys[pygame.K_RIGHT]: x += 2
    canvas.fill((20, 20, 30))
    pygame.draw.rect(canvas, (200,100,80), (x, y, 16, 16))
    screen.blit(pygame.transform.scale(canvas, screen.get_size()), (0,0))
    pygame.display.flip()
    clock.tick(60)
pygame.quit()
```

> Padrões a dominar em Pygame: `Surface` de baixa resolução escalada com
> `pygame.transform.scale`, `pygame.sprite.Group` para colisão, câmera por
> offset aplicado no blit, e parallax com camadas de fundo em velocidades
> diferentes.

---

## 7. Solarus (Lua) & Flare (dados/C++) — engines de gênero pronto

- **Solarus**: engine 2D estilo Zelda (action-adventure top-down). Scripta em
  **Lua**; já traz mapas, heróis, inimigos, itens, HUD e diálogos prontos. Você
  cria mapas no editor e scripta a lógica de cada mapa/inimigo em Lua. Use para
  Zelda-likes — existem jogos completos open-source para estudar como exemplo.
- **Flare** (Free Libre Action RPG Engine): ARPG isométrico estilo Diablo em
  **C++**, com conteúdo definido por **arquivos de dados** (`.txt`/INI) —
  inimigos, itens, poderes, mapas e campanha sem escrever C++. Para um ARPG,
  basta produzir os arquivos de conteúdo sobre o motor.

Use estas quando o gênero do motor casa exatamente com o jogo desejado — você
ganha meses de trabalho.

---

## Regra de seleção rápida

- Sabe a linguagem do usuário? Comece por ela (JS→Phaser, Python→Pyxel/Pygame,
  C→Raylib, Rust→Bevy).
- Quer o caminho mais completo e com mais material desta lista? **Godot 4**.
- O gênero é exatamente Zelda-like ou Diablo-like? **Solarus** / **Flare**.
