# Claude Code via Dev Container — Guia por Linguagem

---

## PARTE 0 — Pré-requisitos (fazer UMA VEZ na máquina)

### 0.1 Docker rodando

```bash
docker ps
```

Se der erro de permissão:

```bash
sudo usermod -aG docker $USER
```

Depois **logout e login** (não basta abrir outro terminal).

Se disser que não conecta ao daemon:

```bash
sudo systemctl start docker
sudo systemctl enable docker
```

### 0.2 Instalar a devcontainer CLI

```bash
npm install -g @devcontainers/cli
devcontainer --version
```

Se der erro de permissão, use `sudo npm install -g @devcontainers/cli`.

### 0.3 Configurar identidade do git no host

Sem isso, commits dentro do container falham.

```bash
git config --global user.name "Seu Nome"
git config --global user.email "seu@email.com"
git config --global --list
```

### 0.4 Criar o atalho de shell

```bash
nano ~/.bashrc
```

Cole no final do arquivo:

```bash
dc() {
  devcontainer up --workspace-folder . >/dev/null && \
  devcontainer exec --workspace-folder . claude "$@"
}

dcsh() {
  devcontainer up --workspace-folder . >/dev/null && \
  devcontainer exec --workspace-folder . bash
}
```

Salvar: `Ctrl+O`, `Enter`, `Ctrl+X`.

```bash
source ~/.bashrc
type dc
```

Deve imprimir a função. Se disser "not found", o texto não foi salvo.

---

## PARTE 1 — Fluxo comum (vale para TODAS as linguagens)

Sempre a partir da **raiz do projeto** (onde está a pasta `.git`).

```bash
cd ~/Documentos/Projetos/ai/nome-do-repo
ls -a          # confirmar que aparece .git
```

### Passo 1 — Criar a pasta de config

```bash
mkdir -p .devcontainer
nano .devcontainer/devcontainer.json
```

### Passo 2 — Colar o JSON da linguagem (ver PARTE 2)

Salvar: `Ctrl+O`, `Enter`, `Ctrl+X`.

### Passo 3 — Subir o container

```bash
devcontainer up --workspace-folder .
```

Primeira vez demora (baixa imagem + instala features). Depois é rápido.

### Passo 4 — Abrir o Claude Code

```bash
devcontainer exec --workspace-folder . claude
```

Ou, com o atalho: `dc`

### Passo 5 — Login (só na primeira vez)

Ele imprime uma URL. Abra no navegador do host, autorize, cole o código de volta no terminal.

### Passo 6 — Não versionar lixo (opcional)

Se NÃO quiser commitar o devcontainer no repo, adicione ao `.gitignore`:

```
.devcontainer/
```

Se quiser compartilhar com o time (recomendado), deixe versionado.

---

## PARTE 2 — Arquivos por linguagem

> Cada bloco abaixo é o conteúdo completo de `.devcontainer/devcontainer.json`.
> **Atenção ao usuário do container** — muda conforme a imagem (`node` ou `vscode`).
> O caminho do mount `.claude` precisa bater com o usuário.

---

### 2.1 Node / TypeScript

Usuário: `node`

```json
{
  "name": "Node + Claude",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:22",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/node/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/node/.claude"
  },
  "remoteUser": "node",
  "postCreateCommand": "npm ci"
}
```

Trocar a versão do Node: `typescript-node:20`, `typescript-node:24`, etc.

Se o projeto usa pnpm ou yarn, trocar o `postCreateCommand` para `pnpm install` ou `yarn install`.

---

### 2.2 Java

Usuário: `vscode`

```json
{
  "name": "Java 21 + Claude",
  "image": "mcr.microsoft.com/devcontainers/java:21",
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "none",
      "installMaven": true,
      "installGradle": true
    },
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/vscode/.claude,type=volume",
    "source=maven-repo,target=/home/vscode/.m2,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode"
}
```

- `"version": "none"` evita instalar um segundo JDK por cima do da imagem.
- Trocar a versão do Java: `devcontainers/java:17`, `java:21`, `java:23`.
- Confirme qual versão o projeto pede no `pom.xml` (`<java.version>` ou `<maven.compiler.release>`).
- O volume `maven-repo` impede rebaixar todas as dependências a cada rebuild.
- Se o projeto tem `./gradlew`, use o wrapper em vez do Gradle da feature.

---

### 2.3 Python

Usuário: `vscode`

```json
{
  "name": "Python 3.12 + Claude",
  "image": "mcr.microsoft.com/devcontainers/python:3.12",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/vscode/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "postCreateCommand": "python -m venv .venv && .venv/bin/pip install -r requirements.txt"
}
```

- Trocar a versão: `python:3.11`, `python:3.12`, `python:3.13`.
- Se o projeto usa Poetry, troque a linha do `postCreateCommand` por `pipx install poetry && poetry install`.
- Se não houver `requirements.txt`, use só `python -m venv .venv`.
- Adicione `.venv/` ao `.gitignore`.

---

### 2.4 Ruby

Usuário: `vscode`

```json
{
  "name": "Ruby 3.3 + Claude",
  "image": "mcr.microsoft.com/devcontainers/ruby:3.3",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/vscode/.claude,type=volume",
    "source=bundle-cache,target=/usr/local/bundle,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "postCreateCommand": "bundle install"
}
```

- Trocar a versão: `ruby:3.2`, `ruby:3.3`, `ruby:3.4`.
- A imagem já traz Ruby compilado — sem espera de build.
- Se não houver `Gemfile`, remova o `postCreateCommand`.

---

### 2.5 Rust

Usuário: `vscode`

```json
{
  "name": "Rust + Claude",
  "image": "mcr.microsoft.com/devcontainers/rust:1",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/vscode/.claude,type=volume",
    "source=cargo-registry,target=/usr/local/cargo/registry,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode"
}
```

- Vem com `rustup` completo: troca de canal, `rustup target add`, `rustup component add clippy`.
- O volume `cargo-registry` evita rebaixar crates a cada rebuild.

---

### 2.6 Projeto com Docker Compose (banco, fila, etc.)

Use quando o projeto já tem `docker-compose.yml` com Postgres, Redis, etc.
É a única forma do Claude conseguir rodar migration e testar ponta a ponta.

**Arquivo 1** — `.devcontainer/docker-compose.dev.yml`:

```yaml
services:
  app:
    build:
      context: ..
      dockerfile: Dockerfile
    volumes:
      - ..:/workspaces/projeto:cached
    command: sleep infinity
```

**Arquivo 2** — `.devcontainer/devcontainer.json`:

```json
{
  "name": "Projeto + Claude",
  "dockerComposeFile": [
    "../docker-compose.yml",
    "docker-compose.dev.yml"
  ],
  "service": "app",
  "workspaceFolder": "/workspaces/projeto",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=claude-code-config,target=/home/node/.claude,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/node/.claude"
  },
  "remoteUser": "node",
  "forwardPorts": [3000, 5432, 6379]
}
```

Ajustar:
- `"service"` deve ser o nome exato do serviço da app no `docker-compose.yml`.
- `"workspaceFolder"` deve bater com o caminho do volume no `docker-compose.dev.yml`.
- `"remoteUser"` depende da imagem do projeto (`node`, `vscode` ou `root`).
- `"forwardPorts"` — as portas que você quer acessar do host.

---

## PARTE 3 — Uso diário

```bash
cd ~/Documentos/Projetos/ai/nome-do-repo
dc
```

Ou sem o atalho:

```bash
devcontainer exec --workspace-folder . claude
```

Shell dentro do container (para rodar `mvn test`, `npm run`, inspecionar):

```bash
dcsh
```

Rebuild após mudar o `devcontainer.json`:

```bash
devcontainer up --workspace-folder . --remove-existing-container
```

---

## PARTE 4 — Pelo editor (alternativa ao terminal)

**Cursor / VS Code:**
1. Instalar a extensão "Dev Containers"
2. `Ctrl+Shift+P` → "Dev Containers: Reopen in Container"
3. Abrir o terminal integrado e rodar `claude`

**IntelliJ:** tem suporte a dev containers, mas é menos estável que VS Code/Cursor. Se travar, use a CLI.

---

## PARTE 5 — Problemas comuns

| Sintoma | Causa | Solução |
|---|---|---|
| `claude: command not found` após rebuild | feature instalada só na criação | trocar `postCreateCommand` por `postStartCommand` |
| Pede login toda vez | volume `.claude` não montado ou caminho errado | conferir se o `target` bate com o usuário (`node` vs `vscode`) |
| Permission denied no `.claude` | usuário do container ≠ dono do volume | `docker run --rm --user root -v claude-code-config:/mnt <imagem> chown -R 1000:1000 /mnt` |
| `git commit` falha | falta user.name/user.email | conferir Passo 0.3 |
| Não acessa o banco | container fora da rede do compose | usar a config da seção 2.6 |
| Porta não abre no navegador | falta forwardPorts | adicionar a porta no `devcontainer.json` |

---

## PARTE 6 — Notas

**Login único vs. por projeto.**
Neste guia todos usam `source=claude-code-config` (nome fixo), então o login é **compartilhado** entre projetos — faz uma vez e vale pra todos.
Se quiser isolar por projeto, troque para `source=claude-code-config-${devcontainerId}` — mas aí faz login em cada um.

**Feature de Node.**
O pacote do Claude Code instala um binário nativo que não usa Node em runtime, então a feature de Node normalmente não é necessária. Se `claude` não for encontrado em alguma imagem, adicione:
```json
"ghcr.io/devcontainers/features/node:1": {}
```

**Segurança.**
O container isola o filesystem, mas **não a rede**. O agente pode fazer `git push`, `curl` para qualquer lugar e usar credenciais montadas. Para sessões com `--dangerously-skip-permissions`, considere o devcontainer de referência da Anthropic (`anthropics/claude-code/.devcontainer`), que inclui `init-firewall.sh` com política default-deny.

**Limpeza.**
```bash
docker volume ls                      # ver o que existe
docker volume rm claude-code-config   # apaga o login
devcontainer up --workspace-folder . --remove-existing-container   # recriar do zero
```
