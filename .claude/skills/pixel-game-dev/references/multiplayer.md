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

Servidor (TypeScript — Schema + Room):
```typescript
import { Room, Client } from "colyseus";
import { Schema, type, MapSchema } from "@colyseus/schema";

export class Player extends Schema {
    @type("number") x = Math.floor(Math.random() * 400);
    @type("number") y = Math.floor(Math.random() * 400);
}

export class State extends Schema {
    @type({ map: Player }) players = new MapSchema<Player>();
    something = "This attribute won't be sent to the client-side";

    createPlayer(sessionId: string) {
        this.players.set(sessionId, new Player());
    }
    removePlayer(sessionId: string) {
        this.players.delete(sessionId);
    }
    movePlayer(sessionId: string, movement: any) {
        const p = this.players.get(sessionId);
        if (movement.x) p.x += movement.x * 10;
        else if (movement.y) p.y += movement.y * 10;
    }
}

export class GameRoom extends Room<State> {
    maxClients = 4;

    onCreate(options: any) {
        this.setState(new State());
        this.onMessage("move", (client, data) => {
            this.state.movePlayer(client.sessionId, data);
        });
    }
    onJoin(client: Client) {
        this.state.createPlayer(client.sessionId);
    }
    onLeave(client: Client) {
        this.state.removePlayer(client.sessionId);
    }
    onDispose() { console.log("Room disposed"); }
}
```
**Pontos-chave do Schema:** propriedades decoradas com `@type` são sincronizadas
automaticamente para os clientes. Propriedades sem `@type` ficam só no servidor
(ex.: `something` acima). Tipos: `"number"`, `"string"`, `"boolean"`, `"int8"`,
`"uint8"`, `"int16"`, `"float32"`, etc. Coleções: `MapSchema`, `ArraySchema`.

Cliente (Phaser/JS):
```js
import { Client } from "colyseus.js";
const client = new Client("ws://localhost:2567");
const room = await client.joinOrCreate("game");
room.onStateChange((state) => { /* render a partir do state */ });
room.state.players.onAdd((player, id) => { /* criar sprite */ });
room.state.players.onRemove((player, id) => { /* destruir sprite */ });
room.send("move", { x: 1, y: 0 });   // envia input
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

### Setup e autenticação (Godot 4)
```gdscript
extends Node

var client: NakamaClient
var session: NakamaSession
var socket: NakamaSocket

func _ready():
    client = Nakama.create_client("defaultkey", "127.0.0.1", 7350, "http")

func authenticate_email(email: String, password: String):
    session = await client.authenticate_email_async(email, password)
    if session.is_exception():
        print("Auth error: ", session.get_exception()._message)
        return
    print(session.token)
    print(session.user_id)
    print(session.username)
    print("Expires at: %s" % session.expire_time)

func authenticate_device():
    session = await client.authenticate_device_async(OS.get_unique_id())
```

### Restaurar sessão
```gdscript
var authtoken = "saved-token-from-storage"
var restored = NakamaClient.restore_session(authtoken)
if restored.expired:
    print("Session expired, must reauthenticate")
```

### Conta e dados
```gdscript
var account = await client.get_account_async(session)
print(account.user.id)
print(account.user.username)
print(account.wallet)
```

### Socket realtime + multiplayer bridge
```gdscript
func connect_realtime():
    socket = Nakama.create_socket_from(client)
    socket.connected.connect(_on_socket_connected)
    socket.closed.connect(_on_socket_closed)
    socket.received_error.connect(_on_socket_error)
    await socket.connect_async(session)

func _on_socket_connected():
    print("Socket connected")

func _on_socket_closed():
    print("Socket closed")

func _on_socket_error(err):
    printerr("Socket error: %s" % err)
```

### Multiplayer com NakamaMultiplayerBridge (integra com MultiplayerAPI do Godot)
```gdscript
var multiplayer_bridge: NakamaMultiplayerBridge

func setup_multiplayer():
    multiplayer_bridge = NakamaMultiplayerBridge.new(socket)
    multiplayer_bridge.match_join_error.connect(_on_match_join_error)
    multiplayer_bridge.match_joined.connect(_on_match_joined)
    get_tree().get_multiplayer().set_multiplayer_peer(multiplayer_bridge.multiplayer_peer)
    # agora use RPCs do Godot normalmente:
    get_tree().get_multiplayer().peer_connected.connect(_on_peer_connected)
    get_tree().get_multiplayer().peer_disconnected.connect(_on_peer_disconnected)

func create_match():
    multiplayer_bridge.create_match()

func join_match(match_id: String):
    multiplayer_bridge.join_match(match_id)

func join_matchmaking():
    var ticket = await socket.add_matchmaker_async()
    if ticket.is_exception():
        print("Matchmaking error: ", ticket.get_exception().message)
        return
    multiplayer_bridge.start_matchmaking(ticket)

func _on_match_joined():
    print("Joined match: ", multiplayer_bridge.match_id)
```

### RPC de movimento (funciona via NakamaMultiplayerBridge)
```gdscript
func _process(delta):
    if not is_multiplayer_authority(): return
    var input = get_input_vector()
    velocity = input * SPEED
    move_and_slide()
    update_remote_position.rpc(position)

@rpc(any_peer)
func update_remote_position(new_position: Vector2):
    position = new_position
```

Use Nakama quando precisa de **infra online de verdade** (contas, save na nuvem,
amigos, ranking, matchmaking) e não só uma sala de partida. O
`NakamaMultiplayerBridge` integra diretamente com a API multiplayer nativa do
Godot, permitindo usar `@rpc` e `multiplayer_peer` como se fosse Godot puro.

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
