# DACOMP GeoGuessr

Jogo estilo [GeoGuessr](https://www.geoguessr.com) pra comunidade acadêmica da
computação da UFSCAR, pelo **DACOMP**: o
jogador vê a foto de um local dentro da UFSCar São Carlos e precisa marcar no mapa
onde acha que ela foi tirada. Quanto mais próximo e rápido o palpite, mais pontos —
e o host acompanha tudo em tempo real.

## Autor(es)

| Autor                   | Curso                   | GitHub                                    |
| ----------------------- | ----------------------- | ----------------------------------------- |
| Maykon dos Santos Gonçalves | Engenharia de Computação | [sgmaykon](https://github.com/sgmaykon) |
| Gustavo Amadeu Mancuzo de Sylos | Ciência da Computação | [Gustag16](https://github.com/Gustag16) |

## Tech Stack

| Camada            | Tecnologia                                                          |
| ----------------- | ------------------------------------------------------------------- |
| Frontend          | React + TypeScript + Vite + Tailwind CSS + Leaflet                  |
| Backend           | Python + Django + Django REST Framework + Django Channels (WebSocket) |
| Banco de dados    | PostgreSQL                                                          |
| Tempo real        | WebSockets (Django Channels / Daphne)                               |
| Deploy            | Nginx + Cloudflare Tunnel (Por conta das limitações do eduroam      |

## Documentação

| Documento                        | Conteúdo                                             |
| -------------------------------- | ---------------------------------------------------- |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Visão geral, comunicação, fluxo da partida, pontuação |
| [docs/API.md](docs/API.md)                 | Endpoints REST                                      |
| [docs/WEBSOCKET.md](docs/WEBSOCKET.md)     | Protocolo WebSocket (ações e eventos)               |
| [docs/DATABASE.md](docs/DATABASE.md)       | Schema do banco e seed                              |
| [deploy/README.md](deploy/README.md)       | Setup de produção (Nginx + Tunnel)                  |

## Estrutura do repositório

```
.
├── backend/
│   └── DACOMP_Guessr/            # Projeto Django (app + API + WebSocket)
├── frontend/
│   └── Guessing_game_frontend/   # Aplicação React (Vite)
├── deploy/
│   └── nginx/                    # Config do Nginx para produção
├── docs/                         # Documentação técnica
├── docker-compose.yaml           # Backend + PostgreSQL
├── init_db.sh                    # Seed inicial das localizações (Postgres)
├── Location_road.csv             # Localizações (fotos de rua)
├── Location_satellite.csv        # Localizações (fotos de satélite)
├── database_schema.txt           # Rascunho antigo (ver docs/DATABASE.md)
└── .env.example                  # Modelo das variáveis do docker-compose
```

## Pré-requisitos

- **Frontend:** Node.js 24+ e npm (via [nvm](https://github.com/nvm-sh/nvm)).
- **Backend + banco:** [Docker](https://www.docker.com/) e Docker Compose.

```bash
# Instalar Node/npm via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 24
node -v   # v24.x
npm -v
```

## Rodando localmente

### 1. Backend + banco (Docker)

Na raiz do repositório:

```bash
cp .env.example .env      # preencha os valores (ver "Variáveis de ambiente")
docker compose up --build
```

Isso sobe o **PostgreSQL** (`:5432`) e o **backend Django** (`:8000`). O
`entrypoint.sh` do container roda automaticamente:

1. `makemigrations` + `migrate`
2. cria o superusuário do admin
3. roda o `seeder` (localizações + sessões iniciais)
4. `runserver 0.0.0.0:8000`

Login do admin (Django Admin em `http://localhost:8000/admin`):
`admin` / `admin123` (configure via `DJANGO_SUPERUSER_*`).

### 2. Frontend

```bash
cd frontend/Guessing_game_frontend
cp .env.example .env      # preencha os valores (ver "Variáveis de ambiente")
npm install
npm run dev
```

A aplicação abre em `http://localhost:5173`.

**Sessões de teste** criadas pelo seed (use o código para entrar na sala): `abcd`,
`SUL1`, `NORT`, `RAND`, `200c`.

### 3. Jogar

1. Abra `http://localhost:5173`, entre na sala com um código acima.
2. Na página de **Host** (`/host`), escolha a sessão e clique para iniciar.
3. Os jogadores palpitam no mapa; ao fim, todos veem o ranking.

> Detalhes do fluxo completo e da pontuação em [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Variáveis de ambiente

### Raiz — `.env` (docker-compose / backend)

| Variável          | Obrigatório | Descrição                          |
| ----------------- | ----------- | ---------------------------------- |
| `SECRET_KEY`      | sim         | Chave secreta do Django            |
| `URL`             | não         | URL base do backend (default `http://127.0.0.1:8000`) |
| `POSTGRES_DB`     | sim         | Nome do banco (default `guessing_game_db`) |
| `POSTGRES_USER`   | sim         | Usuário do banco                   |
| `POSTGRES_PASSWORD`| sim        | Senha do banco                     |
| `POSTGRES_PORT`   | não         | Porta (default `5432`)             |
| `POSTGRES_HOST`   | sim         | Host (em produção, `db` dentro do compose) |

> **Atenção:** as credenciais do banco precisam bater com o serviço `db` do
> `docker-compose.yaml` (que usa `Postgres 15-alpine`). Para rodar o backend fora do
> Docker, há um `.env.example` próprio em `backend/DACOMP_Guessr/DACOMP_Guessr/`.

> A `SECRET_KEY` do Django pode ser gerada com:
> ```bash
> python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
> ```

### Frontend — `.env` (em `frontend/Guessing_game_frontend/`)

| Variável        | Descrição                                          | Default dev            |
| --------------- | -------------------------------------------------- | ---------------------- |
| `VITE_API_URL`  | Base da API REST (usada em dev)                    | `http://127.0.0.1:8000` |
| `VITE_WS_URL`   | Base do WebSocket em dev (**incluir `/ws`**). Usado apenas em dev; em produção é relativo ao mesmo host | `ws://127.0.0.1:8000/ws` |
| `VITE_BACKEND_URL` | URL do backend                                 | `http://127.0.0.1:8000` |
| `DEV`           | Flag de desenvolvimento                             | `False`                |

> `VITE_WS_URL` precisa terminar com `/ws` porque o frontend concatena
> `/lobby/{codigo}/`. Ex.: `VITE_WS_URL=ws://127.0.0.1:8000/ws`.

## Deploy

O deploy usa **Nginx** expondo o frontend buildado + **Cloudflare Tunnel** pra expor
pra internet, com o backend/banco em Docker.

Instruções passo a passo: [deploy/README.md](deploy/README.md).

Resumo:

```bash
# Frontend
cd frontend/Guessing_game_frontend && npm run build
sudo cp -rf dist/* /var/www/dacomp_guessr/
sudo nginx -s reload

# Túnel
cloudflared tunnel --url http://localhost:80

# Backend na mesma máquina
docker compose up
```

> Toda vez que o túnel gerar um novo domínio, adicione-o em `ALLOWED_HOSTS`,
> `CORS_ALLOWED_ORIGINS` e `CSRF_TRUSTED_ORIGINS` em `settings.py`.

## Comandos úteis (backend)

```bash
docker compose exec backend python manage.py seeder            # re-seed
docker compose exec backend python manage.py create_default_admin   # (re)cria admin
docker compose exec backend python manage.py makemigrations    # migrações
docker compose exec backend python manage.py migrate
```

## Seed de localizações

As localizações vêm dos CSVs (`Location_satellite.csv` / `Location_road.csv`). Nos
dois caminhos abaixo o **`Location_satellite.csv`** é o usado:

- **Init do Postgres:** `init_db.sh` (montado em `docker-entrypoint-initdb.d`) copia
  o CSV na primeira subida do banco.
- **Seeder do Django:** `Guessing_Game/management/seeders/seeder_location.py`, que
  lê `Location_satellite.csv` da pasta de seeders.

Para adicionar localizações novas, basta acrescentar linhas no CSV correspondente e
recriar o volume do banco (`docker compose down -v && docker compose up --build`).
