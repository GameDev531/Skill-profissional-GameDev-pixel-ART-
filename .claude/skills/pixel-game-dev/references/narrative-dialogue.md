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
