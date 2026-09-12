# Protocolo WebSocket

O backend usa [Django Channels](https://channels.readthedocs.io/). Todo o protocolo
do jogo (entrada na sala, palpites, rodadas) passa por aqui — as funcionalidades
ficam em `Guessing_Game/consumers.py` (`PlayerConsumer`).

## URL de conexão

```
ws://localhost:8000/ws/lobby/{session_code}/
```

- `session_code` — código de 4 caracteres da sala (ex.: `abcd`).
- Produção: mesmo host do frontend, prefixo `/ws/lobby/{code}/` (servido pelo Nginx).
- Desenvolvimento: `ws://127.0.0.1:8000/ws/lobby/{code}/` — no frontend isso é
  controlado por `VITE_WS_URL` (veja `frontend/Guessing_game_frontend/.env`).

O servidor aceita a conexão se a sessão existir; caso contrário fecha o socket.

## Formato

Todas as mensagens são JSON (`text_data`). As mensagens **client → server** têm um
campo `action`; as mensagens **server → client** têm um campo `type`.

## Ações do cliente

### `join`
Entrar na sala (ou reativar um jogador que já existe).

```json
{
  "action": "join",
  "player": {
    "id": null,
    "nickname": "Maykon",
    "avatar_config": { "head": "redonda", "face": "feliz", "acc": "chapeu", "color": "azul" }
  }
}
```

- `id`: se `null`, cria um novo jogador; se passado e existir na sessão, apenas
  reativa (`is_connected = true`).
- Se a sessão estiver `PLAYING` ou `FINISHED`, entra com `type: error`.
- Resposta: `type: join_success` + broadcast `players_list` para a sala.

### `reconnect`
Reconexão usando o id salvo no `localStorage` do navegador.

```json
{ "action": "reconnect", "player_id": "uuid-do-jogador" }
```

- Resposta: `type: reconnect_success` com os dados da sessão.

### `list_players`
Pede a lista atual de jogadores conectados da sala.

```json
{ "action": "list_players" }
```

- Resposta: `type: players_list`.

### `update_avatar`
Altera nickname/avatar de um jogador e avisa a sala.

```json
{
  "action": "update_avatar",
  "player": { "id": "uuid", "nickname": "NovoNome", "avatar_config": { "..." : "..." } }
}
```

- Broadcast: `type: player_update`.

### `submit_guess`
Envia o palpite do jogador na rodada atual.

```json
{
  "action": "submit_guess",
  "guess": { "latitude": -21.9799, "longitude": -47.8833, "guess_timestamp": 0 }
}
```

- O backend calcula distância/pontuação (ver `docs/ARCHITECTURE.md`), acumula o
  score no `Player` e salva um `Guess`.
- `guess_timestamp` é aceito mas **não é usado** na pontuação (o backend usa
  `session.round_started_at`).
- Se o tempo da rodada acabou (`remaining_time < 0`), o palpite é ignorado.
- Resposta (só para quem enviou): `type: guess_received`.

### `start_round_manual`
Enviada pelo frontend, porém **não existe handler correspondente no consumer**
(fica apenas uma função comentada em `consumers.py`) — não usar. O início do jogo é
feito por REST: `POST /api/sessions/{code}/initialize_rounds/`.

## Eventos do servidor

### `join_success`
```json
{ "type": "join_success", "id": "uuid", "avatar_config": { "..." },
  "message": "Bem-vindo, {nickname}!", "new": true, "nickname": "Maykon" }
```
`new` indica se o jogador foi criado agora (`true`) ou apenas reativado (`false`).

### `reconnect_success`
```json
{ "type": "reconnect_success", "id": "uuid", "nickname": "Maykon",
  "avatar_config": { "..." }, "session_status": "LOBBY",
  "current_round": 0, "total_rounds": 5, "player_score": 0 }
```

### `players_list`
Lista de todos os jogadores da sessão (inclui desconectados, com `is_connected: false`).
```json
{ "type": "players_list",
  "players": [ { "id": "uuid", "nickname": "Maykon",
                 "avatar_config": { "..." }, "is_connected": true } ] }
```

### `player_update`
Um jogador mudou nickname/avatar.
```json
{ "type": "player_update",
  "player": { "id": "uuid", "nickname": "Novo", "avatar_config": { "..." }, "is_connected": true } }
```

### `round_start`
Nova rodada começou.
```json
{ "type": "round_start", "round_number": 1, "message": "Round 1 começou!" }
```

### `time_update`
Countdown da rodada (segundos restantes).
```json
{ "type": "time_update", "round_time": 45 }
```

### `round_timeout`
Fim da rodada. Envia a localização correta, o ranking atual e os palpites.
```json
{ "type": "round_timeout",
  "round_number": 1,
  "message": "Tempo do round 1 esgotado!",
  "correct_lon": -47.88330161948902,
  "correct_lat": -21.979919052176466,
  "players":   [ { "id": "uuid", "nickname": "Maykon", "score": 842,
                    "last_round_score": 842, "avatar_config": { "..." } } ],
  "guesses":   [ { "id": "uuid", "latitude_guess": -22.0, "longitude_guess": -47.9,
                    "distance_in_meters": 2450.5, "points_awarded": 88,
                    "timestamp": "2026-09-11T21:00:00+00:00",
                    "player_id": "uuid", "round_id": 1, "session_id": 1 } ] }
```

### `guess_received`
Confirmação do palpite (só para quem palpitou).
```json
{ "type": "guess_received", "message": "Palpite Recebido.", "score": 88, "total_score": 842 }
```

### `session_status_update`
Mudança de status da sessão. Enviado pelo `update_status` REST e pelo fim do loop
do jogo.
```json
{ "type": "session_status_update", "status": "PLAYING", "message": "O jogo vai começar!", "players": [] }
```
No `FINISHED`, `players` vem preenchido com o ranking final (ordenado por score).

### `error`
```json
{ "type": "error", "message": "A sessão já começou. Não é possível entrar." }
```

## Reconexão

No frontend (`src/api/ws.ts`):
1. Ao abrir o socket, se existe `playerId` no `localStorage`, envia `reconnect`.
2. `shouldReconnect: true` tenta reconectar até 10 vezes (intervalo de 3s).
3. Ao `connect` o backend adiciona o canal ao grupo; `disconnect` marca o jogador
   como `is_connected = false` e agenda a exclusão em 90s (se não reconectar).