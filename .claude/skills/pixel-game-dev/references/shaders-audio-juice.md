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

### Screen shake via Tween (alternativa — sem _process)

Abordagem com `tween_method` — autoconsumível, suporta intensidade + rotação,
não requer variável persistente:

```gdscript
func camera_shake(intensity: float = 1.5, duration: float = 0.4,
        decay: float = 3.0, camera: Camera2D = get_viewport().get_camera_2d()) -> void:
    if camera.has_meta("shake_tween") and camera.get_meta("shake_tween").is_valid():
        camera.get_meta("shake_tween").kill()
    var tween := create_tween()
    camera.set_meta("shake_tween", tween)
    var origin_pos := camera.offset
    var origin_rot := camera.rotation

    var shake_fn := func(progress: float) -> void:
        var remaining := 1.0 - progress
        var strength := intensity * pow(remaining, decay)
        if strength > 0.01:
            camera.offset = origin_pos + Vector2(
                randf_range(-1, 1) * strength * 5.0,
                randf_range(-1, 1) * strength * 5.0)
            camera.rotation = origin_rot + randf_range(-1, 1) * strength * 0.05
        else:
            camera.offset = origin_pos
            camera.rotation = origin_rot

    tween.tween_method(shake_fn, 0.0, 1.0, duration)
    tween.tween_callback(func():
        camera.offset = origin_pos
        camera.rotation = origin_rot)
```

**Quando usar cada:**
- **Trauma (variável):** melhor para múltiplas fontes somando shake continuamente
  (balas, hits rápidos). Decai suavemente a cada frame.
- **Tween (one-shot):** melhor para eventos pontuais (explosão, boss slam). Mais
  simples de chamar, auto-cleanup.

### Hit stop / freeze frame

```gdscript
func hit_stop(duration: float = 0.05) -> void:
    Engine.time_scale = 0.0
    await get_tree().create_timer(duration, true, false, true).timeout
    Engine.time_scale = 1.0
```

Combine: `hit_stop(0.04)` + `camera_shake(2.0, 0.3)` + hit flash = impacto
poderoso. O quarto param `true` no Timer ignora time_scale (processa em real time).

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

### Shader 1 — Palette swap / Hit flash
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

### Shader 2 — Outline (pixel-perfect, 4-sample)
```glsl
shader_type canvas_item;
uniform vec4 line_color : source_color = vec4(1);
uniform float line_thickness : hint_range(0, 10) = 1.0;

void fragment() {
    vec2 size = TEXTURE_PIXEL_SIZE * line_thickness;
    float outline = texture(TEXTURE, UV + vec2(-size.x, 0)).a;
    outline += texture(TEXTURE, UV + vec2(0, size.y)).a;
    outline += texture(TEXTURE, UV + vec2(size.x, 0)).a;
    outline += texture(TEXTURE, UV + vec2(0, -size.y)).a;
    outline = min(outline, 1.0);
    vec4 color = texture(TEXTURE, UV);
    COLOR = mix(color, line_color, outline - color.a);
}
```
Amostra 4 vizinhos cardinais; `line_thickness` controla largura em pixels.
Para outline 8-way (diagonais), adicione 4 amostras extras.

### Shader 3 — Dissolve (noise-based)
```glsl
shader_type canvas_item;
uniform sampler2D noiseTexture : repeat_enable;
uniform float noiseTiling = 2.0;
uniform float dissolveAmount : hint_range(0, 1);
uniform float edgeThickness = 0.01;
uniform vec4 edgeColor : source_color;

void fragment() {
    vec4 originalTexture = texture(TEXTURE, UV);
    vec4 dissolveNoise = texture(noiseTexture, UV * noiseTiling);
    float remappedDissolve = dissolveAmount * (1.01 + edgeThickness) - edgeThickness;
    vec4 step1 = step(remappedDissolve, dissolveNoise);
    vec4 step2 = step(remappedDissolve + edgeThickness, dissolveNoise);
    vec4 edgeArea = step1 - step2;
    edgeArea.a = originalTexture.a;
    originalTexture.a *= step1.r;
    vec4 coloredEdge = edgeArea * edgeColor;
    COLOR = mix(originalTexture, coloredEdge, edgeArea.r);
}
```
Anime `dissolveAmount` 0→1 para "queimar" o sprite. `edgeColor` define a cor da
borda brilhante. Passe uma noise texture (SimplexNoise/FastNoiseLite).

### Shader 4 — Grass sway / wind vertex
```glsl
shader_type canvas_item;
uniform float frequency = 1.0;
uniform float amplitude = 1.0;

void vertex() {
    VERTEX.x += (-VERTEX.y * sin(TIME * frequency) * amplitude);
}
```
Simula vento: desloca X proporcional à altura Y do vértice. Coloque o sprite
com pivot embaixo para a base ficar firme. Funciona para grama, árvores, bandeiras.

### Shader 5 — Circular transition / fade-out
```glsl
shader_type canvas_item;
uniform float fadeAmount : hint_range(0, 1) = 1.0;
uniform float fadeSmoothing : hint_range(0, 1) = 0.5;
uniform vec4 fadeColor : source_color;
uniform vec2 offset = vec2(0.5, 0.5);

void fragment() {
    float dist = distance(UV, offset);
    float circle = smoothstep(fadeAmount, fadeAmount + fadeSmoothing, dist);
    vec4 tex = texture(TEXTURE, UV);
    COLOR = mix(tex, fadeColor, circle);
}
```
Ótimo para transições estilo "buraco negro" (wipe circular). Anime `fadeAmount`.

### Shader 6 — CRT / VHS monitor (pós-processamento)
```glsl
shader_type canvas_item;
uniform bool overlay = false;
uniform float scanlines_opacity : hint_range(0.0, 1.0) = 0.4;
uniform float scanlines_width : hint_range(0.0, 0.5) = 0.25;
uniform float grille_opacity : hint_range(0.0, 1.0) = 0.3;
uniform vec2 resolution = vec2(320.0, 180.0);
uniform bool pixelate = true;
uniform float aberration : hint_range(-1.0, 1.0) = 0.03;
uniform float brightness = 1.4;
uniform float warp_amount : hint_range(0.0, 5.0) = 1.0;
uniform float vignette_intensity = 0.4;
uniform float vignette_opacity : hint_range(0.0, 1.0) = 0.5;

vec2 warp(vec2 uv) {
    vec2 delta = uv - 0.5;
    float delta2 = dot(delta.xy, delta.xy);
    return uv + delta * delta2 * delta2 * warp_amount;
}

float vignette(vec2 uv) {
    uv *= 1.0 - uv.xy;
    float vig = uv.x * uv.y * 15.0;
    return pow(vig, vignette_intensity * vignette_opacity);
}

void fragment() {
    vec2 uv = overlay ? warp(SCREEN_UV) : warp(UV);
    vec2 text_uv = pixelate ? ceil(uv * resolution) / resolution : uv;

    vec4 text;
    text.r = texture(SCREEN_TEXTURE, text_uv + vec2(aberration, 0.0) * 0.1).r;
    text.g = texture(SCREEN_TEXTURE, text_uv - vec2(aberration, 0.0) * 0.1).g;
    text.b = texture(SCREEN_TEXTURE, text_uv).b;
    text.a = 1.0;

    // Grille RGB (simula fósforos do CRT)
    if (grille_opacity > 0.0) {
        float g_r = smoothstep(0.85, 0.95, abs(sin(uv.x * resolution.x * 3.14159)));
        float g_g = smoothstep(0.85, 0.95, abs(sin(1.05 + uv.x * resolution.x * 3.14159)));
        float g_b = smoothstep(0.85, 0.95, abs(sin(2.1 + uv.x * resolution.x * 3.14159)));
        text.r = mix(text.r, text.r * g_r, grille_opacity);
        text.g = mix(text.g, text.g * g_g, grille_opacity);
        text.b = mix(text.b, text.b * g_b, grille_opacity);
    }
    text.rgb = clamp(text.rgb * brightness, vec3(0.0), vec3(1.0));

    // Scanlines
    if (scanlines_opacity > 0.0) {
        float sl = smoothstep(scanlines_width, scanlines_width + 0.5,
            abs(sin(uv.y * resolution.y * 3.14159)));
        text.rgb = mix(text.rgb, text.rgb * vec3(sl), scanlines_opacity);
    }

    text.rgb *= vignette(uv);
    COLOR = text;
}
```
Aplique num `ColorRect` sobre a câmera via `CanvasLayer` (layer alto). Ajuste
`resolution` para a resolução base do jogo (160×144 para GB, 320×180 para
16-bit). O `warp_amount` simula a curvatura do vidro.

### Shader 7 — Water 2D (reflexo + ondulação)
```glsl
shader_type canvas_item;
uniform sampler2D SCREEN_TEXTURE : hint_screen_texture, repeat_enable, filter_nearest;
uniform sampler2D waterNoise : repeat_enable, filter_nearest;
uniform vec4 waterColor : source_color = vec4(0.117, 0.27, 0.58, 1.0);
uniform float colorMix : hint_range(0.0, 1.0) = 0.35;
uniform float distortionForce : hint_range(0.0, 0.1) = 0.01;
uniform float waveBrightness : hint_range(0.0, 3.0) = 1.5;
uniform float waveFreq : hint_range(0.2, 0.9) = 0.6;
uniform float waveSize : hint_range(0.6, 1.2) = 0.9;
uniform vec2 scrollSpeed = vec2(0.1, 0.05);

void fragment() {
    vec2 waterUV = UV;
    waterUV.x += scrollSpeed.x * TIME;
    waterUV.y += cos(TIME * min(1.0, scrollSpeed.y)) * 0.01;

    float noise = texture(waterNoise, waterUV).r;
    waterUV += noise * distortionForce;

    float intensity = smoothstep(waveFreq, waveSize, texture(waterNoise, waterUV).r);
    vec4 color = vec4(waterColor.rgb + intensity * waveBrightness, 1.0);

    vec2 bgUV = SCREEN_UV;
    bgUV += noise * vec2(0.01, 0.01);

    color = mix(texture(SCREEN_TEXTURE, bgUV), color, 0.2);
    COLOR = mix(color, waterColor, colorMix);
}
```
Coloque um `ColorRect` ou `Sprite2D` cobrindo a área de água. A textura de ruído
(FastNoiseLite exportada como textura) controla as ondulações. `scrollSpeed`
anima o fluxo.

---

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
