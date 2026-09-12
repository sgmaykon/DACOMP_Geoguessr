# Banco de dados (PostgreSQL)

Modelos definidos em `backend/DACOMP_Guessr/Guessing_Game/models.py`.
O arquivo `database_schema.txt` na raiz é um rascunho antigo — esta documentação é a
referência atual.

## Relacionamentos

```
Session 1 --- * Round 1 --- 1 Location
   |              |
   1              1
   |              |
   *              *
 Player 1 --- * Guess
 Player 1 --- * Guess (via player)
 Round  1 --- * Guess (via round)
```

## Tabelas

### `Session`
Sessão de jogo (uma sala).

| Campo                | Tipo                          | Descrição                                      |
| -------------------- | ----------------------------- | ---------------------------------------------- |
| `id`                 | PK (auto)                     |                                                |
| `code`               | `CharField(4)` unique         | Código da sala (ex.: `abcd`), gerado automaticamente |
| `name`               | `CharField(50)`               | Nome da sala                                   |
| `total_rounds`       | `IntegerField` default `5`    | Número de rodadas                              |
| `time_limit`         | `IntegerField` default `60`   | Segundos por rodada                            |
| `player_limit`       | `IntegerField` default `10`   | Limite de jogadores                            |
| `current_round_number` | `IntegerField` default `0`  | Rodada atual (0 = lobby)                       |
| `status`             | `CharField` choices           | `INACTIVE` \| `LOBBY` \| `PLAYING` \| `FINISHED` |
| `created_at`         | `DateTimeField` auto          |                                                |
| `round_started_at`   | `DateTimeField` default now   | Referência para o tempo do palpite             |

### `Location`
Uma localização (foto + coordenadas reais).

| Campo        | Tipo               | Descrição                          |
| ------------ | ------------------ | ---------------------------------- |
| `id`         | PK (auto)          |                                    |
| `image_url`  | `URLField`         | URL da foto (Google Drive)         |
| `latitude`   | `FloatField`       | Lat real da foto                   |
| `longitude`  | `FloatField`       | Lon real da foto                   |
| `place_name` | `CharField(100)` blank | Nome do lugar                  |
| `category`   | `CharField(50)` default `"Norte"` | Categoria da localização |

### `Round`
Associa uma `Location` a uma rodada de uma `Session`.

| Campo          | Tipo                | Descrição                            |
| -------------- | ------------------- | ------------------------------------ |
| `id`           | PK (auto)           |                                      |
| `session`      | FK → `Session`      | Sala (rel. `rounds`)                 |
| `location`     | FK → `Location`     | Localização usada (rel. `used_in_rounds`) |
| `round_number` | `IntegerField`      | Número da rodada na sessão           |

Constraints:
- `unique_together = ('session', 'round_number')` — não existem duas rodadas com o
  mesmo número na mesma sessão.
- `ordering = ['round_number']` — vem sempre na ordem.

### `Player`
Jogador de uma sessão.

| Campo             | Tipo                          | Descrição                                   |
| ----------------- | ----------------------------- | ------------------------------------------- |
| `id`              | `UUIDField` PK (default uuid4) | Identificador (usado pelo frontend no localStorage) |
| `session`         | FK → `Session`                | Sala (rel. `players`)                       |
| `nickname`        | `CharField(30)`               | Nome do jogador                             |
| `is_connected`    | `BooleanField` default `True` | Conectado ou não                            |
| `avatar_config`   | `JSONField` default `{}`      | Configuração do avatar (face, chapéu, cor…) |
| `score`           | `FloatField` default `0`      | Pontuação acumulada                          |
| `last_round_score`| `FloatField` default `0`      | Pontuação da última rodada                   |

Formatos de `avatar_config` usados na prática:
- criação: `{ "head": "redonda", "face": "feliz", "acc": "chapeu", "color": "azul" }`
- os campos exatos são definidos no frontend (`PlayerAvatar`).

### `Guess`
Palpite de um jogador em uma rodada.

| Campo                | Tipo                  | Descrição                                   |
| -------------------- | --------------------- | ------------------------------------------- |
| `id`                 | PK (auto)             |                                             |
| `player`             | FK → `Player`         | Jogador (rel. `guesses`)                    |
| `round`              | FK → `Round`          | Rodada (rel. `guesses`)                     |
| `session`            | FK → `Session` (nullável) | Sala (rel. `guesses`)                    |
| `latitude_guess`     | `FloatField`          | Onde o jogador clicou                       |
| `longitude_guess`    | `FloatField`          | Onde o jogador clicou                       |
| `distance_in_meters` | `FloatField`          | Distância (Haversine) calculada pelo backend |
| `points_awarded`     | `IntegerField`        | Pontos ganhos no palpite                    |
| `timestamp`          | `DateTimeField` auto  | Data/hora do palpite (usada p/ desempate)   |

> Os `Guess` são apagados ao final de cada partida (a sessão volta a `LOBBY` com
> scores zerados).

## Seed

Na inicialização (`backend/DACOMP_Guessr/entrypoint.sh`) roda:
1. `makemigrations` / `migrate`
2. `create_default_admin` — superusuário `admin` / `admin123` (configurável via
   `DJANGO_SUPERUSER_*`).
3. `seeder` — popula `Location`, `Session` e `Round` a partir dos CSVs em
   `Guessing_Game/management/seeders/`.

As sessões mais comuns (definidas em `sessions.csv`):

| Código | Nome              | Rodadas | Tempo | Limite |
| ------ | ----------------- | ------- | ----- | ------ |
| abcd   | Sessão Teste      | 5       | 30    | 10     |
| SUL1   | Área Sul          | 9       | 60    | 60     |
| NORT   | Área Norte        | 15      | 60    | 60     |
| RAND   | Sessão Mista      | 20      | 40    | 60     |
| 200c   | SUPER FAST ROUNDS | 48      | 10    | 60     |

> O arquivo `init_db.sh` na raiz também carrega `Location_satellite.csv` direto no
> Postgres na primeira inicialização do container (`docker-entrypoint-initdb.d`).