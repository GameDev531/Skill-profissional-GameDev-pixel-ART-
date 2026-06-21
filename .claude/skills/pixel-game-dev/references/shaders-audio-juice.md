# Shaders, Efeitos, Áudio Retrô & Game Feel

O que transforma um protótipo funcional em algo que "sente bom". Cobre shaders
2D, partículas, screen shake, áudio 8-bit e princípios de *juice*.

---

## 1. Game feel / juice — o conceito

"Juice" é o feedback exagerado a cada ação do jogador. Regra: **toda ação
importante precisa de retorno visual + sonoro imediato.** Pular, acertar,
coletar, morrer — cada um dispara vários efeitos pequenos somados.

Toolbox de feedback de impacto (combine vários por evento):
- **Hit flash:** sprite pisca branco por ~0.05s ao tomar dano (shader de
  substituição de cor).
- **Hit stop / freeze frame:** congela o jogo por 2–4 frames no impacto (dá
  peso).
- **Screen shake:** sacode a câmera proporcional à força (ver abaixo).
- **Knockback:** empurra atacante e atingido em direções opostas.
- **Partículas:** faíscas, poeira ao pousar, fragmentos ao quebrar.
- **Squash & stretch:** escala o sprite no pulo/pouso/impacto.
- **Tween de UI:** números de dano que sobem e somem, barras que "esvaziam"
  animadas.
- **Som + leve variação de pitch** a cada ação (evita repetição robótica).

> Princípio: nenhuma ação "silenciosa". Mas cuidado com excesso — screen shake
> demais enjoa; calibre.

---

## 2. Screen shake (implementação)

Trauma decai com o tempo; o deslocamento é trauma² (suave no fim):
```gdscript
extends Camera2D
var trauma := 0.0
@export var decay := 1.5
@export var max_offset := Vector2(8, 6)

func add_trauma(amount: float) -> void:
    trauma = clamp(trauma + amount, 0.0, 1.0)

func _process(delta: float) -> void:
    if trauma > 0.0:
        trauma = max(trauma - decay * delta, 0.0)
        var s := trauma * trauma
        offset = Vector2(
            max_offset.x * s * randf_range(-1, 1),
            max_offset.y * s * randf_range(-1, 1))
    else:
        offset = Vector2.ZERO
```
Chame `add_trauma(0.4)` ao acertar; `0.8` numa explosão.

---

## 3. Shaders 2D essenciais para pixel

Um shader é um programa que roda na GPU por pixel (fragment) ou por vértice.
Efeitos 2D comuns e quando usar:

| Shader | Para quê |
|---|---|
| **Outline** | Destacar item/inimigo selecionado ou interativo |
| **Hit flash / palette swap** | Piscar branco no dano; trocar paleta (variações de cor de inimigo, dano, times) |
| **Dissolve** | Aparecer/sumir entidades (teleporte, morte, spawn) |
| **Glow / bloom** | Brilho em fontes de luz, projéteis, lava |
| **Water (top-down e side)** | Distorção/ondulação de água |
| **Reflection** | Reflexo em água/espelho |
| **Cloud shadows** | Sombras de nuvem baseadas em ruído sobre o mundo |
| **X-ray mask** | Mostrar o player atrás de paredes (silhueta) |
| **Pixelation / CRT / scanlines** | Estética retrô de tela; pós-processamento |
| **Screen distortion / shockwave** | Onda de choque em explosões |
| **2D lighting / normal maps** | Luz dinâmica, dia/noite, tochas |

Exemplo — palette swap / hit flash (Godot, canvas_item shader):
```glsl
shader_type canvas_item;
uniform float flash : hint_range(0,1) = 0.0;
uniform vec4 flash_color : source_color = vec4(1.0);
void fragment() {
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(tex, vec4(flash_color.rgb, tex.a), flash * tex.a);
}
```
Anime o uniform `flash` 1→0 com um tween ao tomar dano.

Organização típica de uma biblioteca de shaders: pasta `Shaders/` com o `.gdshader`
e `Demos/` com cenas mostrando cada efeito aplicado — estude o demo e copie o
shader. Para efeitos de tela inteira (CRT, vinheta, color grading), aplique como
pós-processamento num `CanvasLayer`/viewport.

Partículas e efeitos de câmera (zoom punch, flash de tela, vinheta de dano)
complementam shaders: use o sistema de partículas 2D da engine para poeira,
faíscas, fumaça e fragmentos, disparados nos mesmos eventos do feedback.

---

## 4. Áudio retrô (SFX 8-bit) — geração procedural

Som retrô é gerado por síntese (sem gravar). O padrão **sfxr/jsfxr** gera efeitos
8-bit a partir de presets e parâmetros — perfeito para pixel.

Presets prontos (cada um cobre um evento clássico):
`pickupCoin`, `laserShoot`, `explosion`, `powerUp`, `hitHurt`, `jump`,
`blipSelect`, `synth`, `tone`, `click`, `random`.

Uso em JS:
```js
import { sfxr } from "jsfxr";
const coin = sfxr.generate("pickupCoin");  // gera params do preset
sfxr.play(coin);                           // toca
```
Som customizado a partir de parâmetros:
```js
var sound = {
  wave_type: 1,          // 0=square,1=sawtooth,2=sine,3=noise
  p_env_attack: 0, p_env_sustain: 0.317, p_env_decay: 0.271,
  p_base_freq: 0.261, sound_vol: 0.25, sample_rate: 44100, sample_size: 8
};
var audio = sfxr.toAudio(sound);
audio.play();
```
Parâmetros controlam **forma de onda**, **envelope** (attack/sustain/decay),
**frequência/modulação**, **filtros** e **volume**. Fluxo recomendado: gere e
ajuste no editor sfxr visual, exporte os params/`.wav`, embuta no jogo.

Para engines não-JS, gere os `.wav` no editor sfxr e carregue como áudio normal.
Para **música**, use trackers/chiptune (formatos mod/xm) ou os bancos de som
embutidos (ex.: editor de som do Pyxel) com canais limitados para manter a
estética.

### Boas práticas de áudio
- **Variação de pitch** (±5–10%) a cada disparo evita fadiga.
- **Mixagem por barramentos:** SFX, música e UI em buses separados com volume
  independente.
- **Ducking:** abaixe a música levemente quando um SFX importante toca.
- **Limite de vozes:** não toque 50 cópias do mesmo som no mesmo frame.

---

## 5. Transições e polish de tela

- **Fade in/out** entre cenas; **wipe**/dissolve para trocas de fase.
- **Flash de tela** branco/vermelho em eventos fortes (curto!).
- **Vinheta de dano** vermelha nas bordas com HP baixo.
- **Slow-motion** breve em mortes/finalizações.
- **Camera punch/zoom** rápido no impacto, voltando suave.

---

## Checklist de "sente bom"

- [ ] Toda ação principal tem som + efeito visual imediatos.
- [ ] Dano dispara hit flash + screen shake + knockback + som.
- [ ] Coleta/power-up tem partícula + jingle.
- [ ] Pulo/pouso com squash & stretch e poeira.
- [ ] Transições de cena com fade; sem cortes secos.
- [ ] SFX com variação de pitch; buses de áudio separados.
- [ ] Juice calibrado (forte, mas sem enjoar).
