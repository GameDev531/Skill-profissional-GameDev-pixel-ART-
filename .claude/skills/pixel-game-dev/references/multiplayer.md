# Multiplayer & Backend

Como adicionar multiplayer em tempo real e serviços online (contas, save na
nuvem, matchmaking, placares). Duas abordagens: **servidor autoritativo de sala**
(Colyseus, para web/Phaser) e **backend social/jogo completo** (Nakama, com
cliente Godot).

---

## 1. Conceitos antes de codar

- **Autoridade:** prefira **servidor autoritativo** — o servidor calcula o estado
  real e os clientes apenas enviam inputs e renderizam. Evita trapaça e
  divergência. Clientes "puro peer-to-peer confiando no cliente" são fáceis de
  burlar.
- **Sincronização de estado:** o servidor mantém o estado da sala e envia
  *patches* (só o que mudou) aos clientes.
- **Client-side prediction + reconciliation:** o cliente prevê o próprio
  movimento localmente (sem esperar o servidor) e corrige quando o estado
  autoritativo chega. Essencial para ação responsiva com latência.
- **Interpolação:** renderize entidades remotas interpolando entre os últimos
  estados recebidos (suaviza a ~20 ticks/s do servidor para 60fps).
- **Tick rate:** servidor roda a simulação num passo fixo (ex.: 20–30Hz) e
  transmite snapshots; não envie a cada frame.

---

## 2. Colyseus — salas autoritativas em tempo real (Node/JS)

Modelo: o servidor define **Rooms**; cada sala tem um **state** sincronizado
automaticamente (Schema) e recebe **mensagens** dos clientes. Ótimo com Phaser.

Servidor (sala):
```js
import { Room } from "colyseus";
import { Schema, MapSchema, type } from "@colyseus/schema";

class Player extends Schema {}
type("number")(Player.prototype, "x");
type("number")(Player.prototype, "y");

class State extends Schema {}
type({ map: Player })(State.prototype, "players");

export class GameRoom extends Room {
  onCreate() {
    this.setState(new State());
    this.setSimulationInterval((dt) => this.update(dt), 1000/20); // 20Hz
    this.onMessage("move", (client, input) => {
      const p = this.state.players.get(client.sessionId);
      p.x += input.dx; p.y += input.dy;   // servidor é autoritativo
    });
  }
  onJoin(client) {
    const p = new Player(); p.x = 100; p.y = 100;
    this.state.players.set(client.sessionId, p);
  }
  onLeave(client) { this.state.players.delete(client.sessionId); }
  update(dt) { /* física/IA do servidor */ }
}
```
Cliente (Phaser):
```js
import { Client } from "colyseus.js";
const client = new Client("ws://localhost:2567");
const room = await client.joinOrCreate("game");
room.onStateChange((state) => { /* render a partir do state */ });
room.state.players.onAdd((player, id) => { /* criar sprite */ });
room.send("move", { dx: 1, dy: 0 });   // envia input
```
Padrão: cliente envia **inputs**, servidor muda o **state**, mudanças propagam
via `onStateChange`/callbacks de coleção (`onAdd`/`onRemove`).

---

## 3. Nakama — backend social/jogo completo (com cliente Godot)

Nakama é um servidor open-source que entrega serviços online prontos, com SDK
cliente para Godot (e outras engines):
- **Autenticação:** device, email, social (login de jogadores).
- **Storage:** save na nuvem por usuário (objetos JSON com permissões).
- **Realtime multiplayer:** matches autoritativas (lógica em módulos
  server-side Lua/Go/TS) ou relayed (peer via servidor).
- **Matchmaker:** parear jogadores por critérios (skill, modo).
- **Social:** amigos, grupos/clãs, chat em tempo real, presença.
- **Leaderboards & tournaments:** placares e torneios.
- **RPCs:** funções custom no servidor chamadas pelo cliente.

Fluxo típico no Godot:
```gdscript
var client := Nakama.create_client("defaultkey", "127.0.0.1", 7350, "http")
var session := await client.authenticate_device_async(OS.get_unique_id())
# socket realtime:
var socket := Nakama.create_socket_from(client)
await socket.connect_async(session)
# entrar/criar match, enviar/receber dados de estado, escutar presença...
```
Use Nakama quando precisa de **infra online de verdade** (contas, save na nuvem,
amigos, ranking, matchmaking) e não só uma sala de partida.

---

## 4. Escolha rápida

| Preciso de… | Use |
|---|---|
| Partida em tempo real num jogo web/Phaser | **Colyseus** (salas autoritativas) |
| Contas, save na nuvem, amigos, ranking, matchmaking (Godot) | **Nakama** |
| Ambos (sala de jogo + serviços sociais) | Nakama p/ serviços + match autoritativa |

Dicas finais:
- Comece **single-player sólido**; multiplayer multiplica a complexidade.
- Projete o estado para ser **pequeno e serializável** desde o início.
- Nunca confie em dados do cliente para coisas que afetam outros jogadores.
- Teste com latência artificial (simule 100–200ms) cedo.
