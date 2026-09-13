# Installation

Obelisk separates framework source from the server runtime stack:

- [`framework`](https://github.com/Obelisk-Framework/framework) contains the core FiveM resource, Vue NUI, tests, and CLI.
- [`infrastructure`](https://github.com/Obelisk-Framework/infrastructure) contains Docker Compose, FXServer setup, and the server configuration template.
- [`oblsk_connector`](https://github.com/Obelisk-Framework/oblsk_connector) bridges the Lua ORM to MySQL/MariaDB or PostgreSQL.

Choose framework development if you only need the CLI and tests. Use the infrastructure path to run an actual FiveM server.

## Framework development

### Requirements

- Git
- Node.js and npm
- Lua 5.4 for the test suite

Clone the framework and install its dependencies:

```bash
git clone https://github.com/Obelisk-Framework/framework.git
cd framework
npm install
node cli/index.js --help
```

To invoke the CLI as `obelisk`, link it globally:

```bash
chmod +x cli/index.js
npm link
obelisk --help
```

The package also exposes generators as npm scripts, including `make:module`, `make:plugin`, `make:model`, `make:migration`, `make:seeder`, `make:action`, `make:interaction`, and `make:policy`.

Generated files are written relative to the current working directory. Run generators from the framework repository root.

```bash
npm run cli -- make:module MyFeature
npm run cli -- registry:generate
npm test
```

## Run a FiveM server with Docker

### Requirements

- Git
- Docker with Docker Compose
- Node.js 20+ and npm for `oblsk_connector`
- A FiveM server license key from [Cfx.re Keymaster](https://keymaster.fivem.net/)

### 1. Clone the infrastructure and resources

```bash
git clone https://github.com/Obelisk-Framework/infrastructure.git obelisk
cd obelisk

git clone https://github.com/Obelisk-Framework/framework.git core
git clone https://github.com/Obelisk-Framework/oblsk_connector.git oblsk_connector
npm ci --prefix oblsk_connector
```

Keep the resource directory names exactly as shown. Docker Compose mounts `./core` and `./oblsk_connector` into FXServer under those resource names.

### 2. Configure the server

```bash
cp server.cfg.example server.cfg
```

Open `server.cfg` and replace the placeholder `sv_licenseKey "changeme"` with your real server license key. The local `server.cfg` is ignored by Git.

The container downloads the recommended Linux FXServer artifact automatically if `fxserver/` is empty. To download or update it explicitly on Linux or in WSL, run:

```bash
./scripts/update-fivem-server.sh
```

The script requires `curl`, `jq`, and `tar` with XZ support.

### 3. Start with MariaDB

```bash
docker compose --profile mariadb up --build mariadb fxserver
```

Both database services use Compose profiles, so the profile flag is required. This command starts MariaDB and FXServer; the FXServer entrypoint also starts the connector's Node.js sidecar.

Open txAdmin at `http://localhost:40120`. FiveM traffic is exposed on TCP and UDP port `30120`.

## Switch to PostgreSQL

In `server.cfg`, set both the PostgreSQL connection string and driver:

```cfg
set mysql_connection_string "postgres://obelisk:obelisk_password@postgres:5432/fivem"
set db_driver "postgres"
```

Then start the PostgreSQL profile:

```bash
docker compose --profile postgres up --build postgres fxserver
```

Do not start MariaDB alongside it. The profile controls which database container runs, while `db_driver` selects the ORM's SQL dialect. See [ORM: Dialects](/concepts/orm#dialects) for details.

## Update an installation

From the infrastructure repository root:

```bash
git pull --ff-only
git -C core pull --ff-only
git -C oblsk_connector pull --ff-only
npm ci --prefix oblsk_connector
./scripts/update-fivem-server.sh
docker compose --profile mariadb up --build -d mariadb fxserver
```

Replace `mariadb` with `postgres` when using PostgreSQL. Stop FXServer before replacing an existing FXServer artifact.

For additional commands and troubleshooting, see the [infrastructure README](https://github.com/Obelisk-Framework/infrastructure#readme).
