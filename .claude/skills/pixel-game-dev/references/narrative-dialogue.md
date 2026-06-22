# Diálogo & Narrativa Ramificada

Como escrever diálogo, escolhas e histórias com estado. Três sistemas maduros:
**Ink** (prosa/narrativa interativa), **Yarn Spinner** (diálogo estilo roteiro)
e **Dialogic** (visual, integrado ao Godot).

---

## 1. Ink — narrativa interativa (linguagem de roteiro)

Ink é uma linguagem de marcação para histórias ramificadas. O texto é o
conteúdo; marcações controlam fluxo, escolhas, variáveis e lógica. Integra via
runtime (C#/Unity, e ports) — você roda o `.ink` compilado e recebe linhas +
escolhas.

### Conteúdo e comentários
```ink
Hello, world!     // isto é uma fala
/* comentário
   de bloco */
```

### Escolhas
```ink
*  [Once-only]  aparece só uma vez (marcada com *)
+  [Sticky]     pode ser reescolhida (marcada com +)
```
Colchetes controlam o que aparece: texto **antes** de `[]` aparece na escolha E
na saída; **dentro** de `[]` só na escolha; **depois** só na saída.
```ink
*  Hello [back!] right back to you!
   Nice to hear from you!
```

### Knots, stitches e diverts (estrutura e fluxo)
```ink
=== back_in_london ===          // knot (seção)
We arrived into London at 9.45pm.
-> the_orient_express           // divert (vai para)

=== the_orient_express ===
= in_first_class                // stitch (sub-seção)
   ...
-> the_orient_express.in_third_class   // endereço por ponto
-> END                          // fim da história
```

### Glue (sem quebra de linha)
```ink
We hurried home <>
-> to_savile_row    // "We hurried home to Savile Row"
```

### Variáveis e lógica
```ink
VAR knows = false
VAR name = "Emilia"
~ knows = true            // ~ atribui/executa
~ x = RANDOM(1, 6)        // POW, RANDOM, +,-,*,/,%
{ x == 1: ... }
```

### Escolhas condicionais
```ink
*  { not visit_paris } [Ir a Paris] -> visit_paris
+  { visit_paris }      [Voltar a Paris] -> visit_paris
// combine com and/&&, or/||, not
```

### Texto condicional e alternativas
```ink
{met_blofeld: "Eu o vi." | "Não o encontrei."}     // if/else inline
The radio hissed. {"Três!"|"Dois!"|"Um!"|Silêncio.} // sequência
It was {&Seg|Ter|Qua} hoje.                          // ciclo (&)
{!Eu ri.|Eu sorri.}                                   // once-only (!)
Joguei a moeda. {~Cara|Coroa}.                        // shuffle (~)
```

### Blocos condicionais e switch
```ink
{ x > 0:
   ~ y = x - 1
- else:
   ~ y = x + 1
}
{ x:
- 0:    zero
- 1:    one
- else: lots
}
```

### Gathers e weave (juntar ramos)
```ink
*  "Estou cansado[."]," repeti.
   "Sério," respondeu ele.
*  "Nada, Monsieur!"[]
   "Muito bem."
-  Com isso, Monsieur Fogg saiu da sala.   // gather: '-' reúne os ramos
```
Aninhe com múltiplos `*` e `-` (weave). Rotule pontos com `(label)` para testar
depois (`{label}`).

### Tunnels e threads
```ink
-> crossing ->        // tunnel: sub-rotina que retorna
=== crossing ===
...
->->                  // retorna ao chamador

<- conversation       // thread: injeta outra seção no fluxo atual
-> DONE               // fim de um thread (≠ -> END)
```

### Funções e listas
```ink
=== function lerp(a, b, k) ===
~ return ((b - a) * k) + a

LIST estado = frio, fervendo, quente
VAR chaleira = frio
~ chaleira = fervendo
{ chaleira == fervendo: A chaleira está quente. }
// LISTs multivaloradas: += / -= / '?' (contém); LIST_COUNT/MIN/MAX/ALL/INVERT
```

Consultas de jogo: `CHOICE_COUNT()`, `TURNS()`, `TURNS_SINCE(-> knot)`,
`SEED_RANDOM(n)` (fixar aleatório p/ teste).

Use Ink quando a narrativa é o coração do jogo (escolhas, ramificação densa,
estado de história complexo) e você quer escrever em arquivos de texto.

---

## 2. Yarn Spinner — diálogo estilo roteiro

Yarn organiza diálogo em **nós (nodes)**. Quando roda, o *Dialogue Runner*
entrega três coisas ao jogo: **Lines** (falas), **Options** (escolhas) e
**Commands** (fazer algo na cena). Formato simples, amigável para escritores.

### Estrutura de nó
```yarn
title: Start
---
Guarda: Alto! Quem vem lá?
Você: Só um viajante.
-> Sou amigo.
    Guarda: Então passe.
-> Quero passar à força. <<if $forte>>
    Guarda: Você não passará!
    <<jump Combate>>
===
```
- Cabeçalho com `title:` (e outros headers) → `---` inicia o corpo → `===` fecha.
- Falas: `Personagem: texto` (o nome antes de `:` é o falante).
- **Opções:** linhas começando com `->`; o conteúdo indentado abaixo roda se
  escolhida.

### Comandos, variáveis e lógica
```yarn
<<declare $gold = 0>>
<<set $gold = $gold + 10>>
<<if $gold > 5>>
  Mercador: Você pode comprar isto.
<<elseif $gold == 0>>
  Mercador: Sem ouro, sem conversa.
<<else>>
  Mercador: Quase lá.
<<endif>>
<<jump OutroNo>>          // pula para outro nó
<<play_sound "coin">>     // comando custom que VOCÊ registra no jogo
```
- `$var` são variáveis; `<<set>>`/`<<declare>>` definem.
- `<<...>>` são comandos: alguns embutidos (`jump`, `stop`, `wait`), outros
  **você registra** no runner para acionar lógica do jogo (tocar som, dar item,
  mover câmera).
- Texto interpolado: `{$playerName}`. Markup: `[b]...[/b]` e tags custom.

Use Yarn quando quer diálogo de personagens (visual novel, RPG, aventura) com
sintaxe enxuta e integração forte com a cena/engine via comandos. Existem
runners para várias engines, inclusive uma implementação em GDScript puro para
Godot.

---

## 3. Dialogic — diálogo visual integrado ao Godot

Plugin de Godot 4 para narrativa, focado em **editor visual** (sem precisar
escrever sintaxe). Conceitos:
- **Timeline:** sequência de **eventos** (falas, escolhas, mudar personagem,
  esperar, set de variável, chamar função). É a unidade de diálogo.
- **Characters & Portraits:** dados de personagem com retratos; troca de
  expressão/pose durante a fala.
- **Text events:** o evento principal (fala com efeito de digitação).
- **Choices:** ramificam a timeline conforme a decisão do jogador.
- **Variables:** estado do jogo (flags de quest, relacionamento) consultável nas
  condições.
- **Glossary:** termos com definição acessível em runtime.

Iniciar por código:
```gdscript
Dialogic.start("minha_timeline")   # começa o diálogo
Dialogic.VAR.gold = 10             # ler/escrever variáveis
Dialogic.signal_event.connect(_on_dialogic_signal)  # eventos custom da timeline
```

Use Dialogic quando o jogo é Godot e você quer montar diálogo/visual novel
rápido num editor, com retratos e eventos, sem aprender uma linguagem.

---

## 4. Como escolher

| Situação | Sistema |
|---|---|
| Narrativa ramificada densa, escolhas com estado, multi-engine | **Ink** |
| Diálogo de personagens com comandos que afetam a cena | **Yarn Spinner** |
| Projeto Godot, quer editor visual + retratos + visual novel | **Dialogic** |

Boas práticas comuns:
- **Separe conteúdo de código:** texto/ramos em arquivos de diálogo, lógica de
  jogo nos comandos/callbacks.
- **Variáveis de história** dirigem escolhas condicionais e finais.
- **Localização:** todos esses sistemas extraem strings para tradução — planeje
  IDs de linha desde o início.
- **Teste com seeds fixas** (Ink `SEED_RANDOM`) para reproduzir ramos.

---

## 5. Caixa de diálogo com efeito typewriter (implementação prática)

Padrão completo para caixa de diálogo pixel-art com efeito de digitação,
retrato de personagem e avanço por input.

### Estrutura de cena
```
DialogueBox (Control — anchors bottom-wide)
├── NinePatchRect (fundo da caixa — texture pixel-art, margins configuradas)
│   ├── MarginContainer
│   │   ├── HBoxContainer
│   │   │   ├── Portrait (TextureRect — 48×48 ou 64×64)
│   │   │   └── VBoxContainer
│   │   │       ├── NameLabel (Label — pixel font, bold)
│   │   │       └── DialogueLabel (RichTextLabel — pixel font)
│   └── AdvanceIndicator (TextureRect — seta animada, bottom-right)
├── AnimationPlayer (para AdvanceIndicator pulsando)
└── AudioStreamPlayer (sfx de digitação — blip curto)
```

### Script completo

```gdscript
class_name DialogueBox
extends Control

signal dialogue_finished
signal line_finished

@export var char_speed: float = 0.03
@export var punctuation_pause: float = 0.12

@onready var portrait: TextureRect = %Portrait
@onready var name_label: Label = %NameLabel
@onready var dialogue_label: RichTextLabel = %DialogueLabel
@onready var advance_indicator: TextureRect = %AdvanceIndicator
@onready var blip_player: AudioStreamPlayer = %BlipPlayer

var _lines: Array[Dictionary] = []
var _current_line: int = -1
var _tween: Tween
var _is_typing := false
var _is_active := false

func start_dialogue(lines: Array[Dictionary]) -> void:
    _lines = lines
    _current_line = -1
    _is_active = true
    visible = true
    advance_indicator.visible = false
    _advance()

func _advance() -> void:
    _current_line += 1
    if _current_line >= _lines.size():
        _close()
        return
    var entry: Dictionary = _lines[_current_line]
    name_label.text = entry.get("name", "")
    if entry.has("portrait"):
        portrait.texture = entry["portrait"]
        portrait.visible = true
    else:
        portrait.visible = false
    dialogue_label.text = entry.get("text", "")
    dialogue_label.visible_ratio = 0.0
    advance_indicator.visible = false
    _type_text()

func _type_text() -> void:
    _is_typing = true
    var total_chars := dialogue_label.get_total_character_count()
    if total_chars == 0:
        _finish_line()
        return
    var duration := total_chars * char_speed
    if _tween:
        _tween.kill()
    _tween = create_tween()
    _tween.tween_property(dialogue_label, "visible_ratio", 1.0, duration)
    _tween.tween_callback(_finish_line)
    _start_blip_timer(total_chars)

func _start_blip_timer(total_chars: int) -> void:
    for i in range(total_chars):
        var delay := i * char_speed
        var char := _get_char_at(i)
        if char in [".", ",", "!", "?", ";"]:
            delay += punctuation_pause
        get_tree().create_timer(delay).timeout.connect(
            func(): if _is_typing: blip_player.play()
        )

func _get_char_at(index: int) -> String:
    var stripped := dialogue_label.get_parsed_text()
    if index < stripped.length():
        return stripped[index]
    return ""

func _finish_line() -> void:
    _is_typing = false
    dialogue_label.visible_ratio = 1.0
    advance_indicator.visible = true
    line_finished.emit()

func _skip_typing() -> void:
    if _tween:
        _tween.kill()
    _finish_line()

func _close() -> void:
    _is_active = false
    visible = false
    dialogue_finished.emit()

func _unhandled_input(event: InputEvent) -> void:
    if not _is_active:
        return
    if event.is_action_pressed("ui_accept") or event.is_action_pressed("interact"):
        get_viewport().set_input_as_handled()
        if _is_typing:
            _skip_typing()
        else:
            _advance()
```

### Uso

```gdscript
var lines := [
    {"name": "Mago", "portrait": preload("res://assets/portraits/mago.png"),
     "text": "O cristal está se fragmentando... precisamos agir rápido."},
    {"name": "Guerreira", "portrait": preload("res://assets/portraits/guerreira.png"),
     "text": "Deixa comigo. Cobre minha retaguarda!"},
    {"name": "", "text": "O chão tremeu sob seus pés..."},
]
$DialogueBox.start_dialogue(lines)
await $DialogueBox.dialogue_finished
# continua gameplay
```

### Variações e dicas

- **Pausa em pontuação:** o timer extra em `.` `,` `!` `?` cria ritmo natural.
- **BBCode no RichTextLabel:** use `[color]`, `[wave]`, `[shake]` para ênfase
  inline — `visible_ratio` respeita tags BBCode automaticamente.
- **Múltiplas velocidades:** segure um botão para 3× a velocidade
  (`char_speed / 3`).
- **Escolhas inline:** ao final de uma linha com `choices`, instancie botões
  dentro do VBox e conecte ao sistema de narrative (Ink/Yarn).
- **NinePatchRect pixel-perfect:** configure `patch_margin` nos 4 lados (ex.: 8px
  se a borda da textura tem 8px), e use `axis_stretch_mode = TILE` para manter
  pixel crisp.
- **Integração com Ink/Yarn:** substitua o array de dicts por um parser que lê
  linhas do Ink runner ou Yarn dialogue runner e popula `_lines` dinamicamente.
