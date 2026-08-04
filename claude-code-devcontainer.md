# Claude Code + Dev Container — Guia Explicado

> Versão corrigida, baseada no que quebrou na prática.
> Substitui o guia anterior.

---

# PARTE A — O que estamos montando e por quê

## O problema

Rodar o Claude Code direto na máquina significa dar a ele acesso ao teu sistema inteiro. Rodar dentro de um container isola o sistema de arquivos: ele só enxerga o projeto.

Mas um container isolado tem um efeito colateral: ele também não enxerga o banco de dados do projeto. E sem banco, o agente não consegue rodar migration, teste de integração, nem subir a aplicação — vira um container que só escreve texto.

## A solução

O **dev container** resolve os dois de uma vez. Ele sobe junto com os serviços do `docker-compose.yml` do projeto (Postgres, Redis, etc.), na mesma rede. O Claude fica isolado da tua máquina, mas conectado ao ambiente do projeto.

## As peças

| Peça | O que faz | Onde vive |
|---|---|---|
| `devcontainers/cli` | comando que sobe o container | instalado uma vez na máquina |
| `.devcontainer/devcontainer.json` | descreve o ambiente | no projeto |
| `.devcontainer/docker-compose.dev.yml` | acrescenta o serviço da app ao compose do time | no projeto |
| feature `claude-code` | instala o Claude Code dentro do container | baixada no build |
| bind de `~/.claude` | login + `CLAUDE.md` de usuário + config de MCP, vindos do host | pasta real na tua máquina |
| volumes nomeados | cache de dependências entre execuções | no Docker |
| função `dc` no shell | atalho para subir + abrir | no `~/.bashrc` |

---

# PARTE B — Setup da máquina (UMA VEZ)

## Etapa B1 — Docker funcionando

````bash
docker ps
````

**Por quê:** tudo depende do daemon do Docker estar rodando e do teu usuário ter permissão de falar com ele.

Se der erro de permissão:
````bash
sudo usermod -aG docker $USER
````
Depois **logout e login** — abrir outro terminal não basta, porque o grupo só é reavaliado no login.

Se disser que não conecta ao daemon:
````bash
sudo systemctl start docker
sudo systemctl enable docker
````
O `enable` faz o Docker subir junto com o sistema, para você não repetir isso a cada reboot.

---

## Etapa B2 — npm global sem sudo

````bash
mkdir -p ~/.npm-global
npm config set prefix ~/.npm-global
echo 'export PATH="$HOME/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
````

**Por quê:** por padrão o npm instala pacotes globais em `/usr/lib/node_modules`, que pertence ao root. Sem isso, todo `npm install -g` exige sudo e deixa arquivos root-owned no teu home, que dão problema depois.

---

## Etapa B3 — Instalar a CLI

````bash
npm install -g @devcontainers/cli
devcontainer --version
````

**Por quê:** é ela que lê o `devcontainer.json`, mescla os arquivos compose, faz o build com as features e sobe tudo. Sem ela você dependeria do VS Code para usar dev containers.

---

## Etapa B4 — Identidade do git

````bash
git config --global user.name "Seu Nome"
git config --global user.email "seu@email.com"
````

**Por quê:** o container herda o `.gitconfig` do host. Sem `user.name` e `user.email`, qualquer `git commit` feito lá dentro falha.

---

## Etapa B5 — Atalhos no shell

````bash
nano ~/.bashrc
````

Cole no final:

````bash
dc() {
  devcontainer up --workspace-folder . >/dev/null && \
  devcontainer exec --workspace-folder . claude "$@"
}

dcsh() {
  devcontainer up --workspace-folder . >/dev/null && \
  devcontainer exec --workspace-folder . bash
}
````

`Ctrl+O`, `Enter`, `Ctrl+X`, depois:

````bash
source ~/.bashrc
type dc
````

**Por quê:** `dc` abre o Claude, `dcsh` abre um shell dentro do container. O `devcontainer up` antes de cada um é barato — se o container já está de pé, ele só reaproveita.

**Importante:** rode `dc` e `dcsh` sempre da **raiz de um projeto**. Eles montam o diretório atual; da tua home, montariam tudo.

---

# PARTE C — Setup por projeto

Exemplo real: projeto Java 17 + Maven + Spring Boot + Postgres via compose.

## Etapa C1 — Descobrir o que o projeto pede

````bash
cd ~/caminho/do/projeto
ls -a                                    # confirma que tem .git
git branch --show-current                # confirma a branch
ls | grep -E "pom.xml|build.gradle"      # Maven ou Gradle?
grep -E "java.version|maven.compiler" pom.xml
ls | grep -E "docker-compose|compose.yml"
grep -E "datasource|jpa|flyway" src/main/resources/application*
````

**Por quê:** a imagem do container tem que bater com a versão que o projeto compila. E se houver banco, a configuração muda (Parte C3 em vez de C2).

Anote:
- versão do Java (ex: `17`)
- se tem `docker-compose.yml`
- o nome do serviço do banco no compose (ex: `db`)
- para onde o `application.properties` aponta (ex: `localhost:5432`)

---

## Etapa C2 — SEM banco (projeto simples)

````bash
mkdir -p .devcontainer
nano .devcontainer/devcontainer.json
````

````json
{
  "name": "nome-do-projeto",
  "image": "mcr.microsoft.com/devcontainers/java:17",
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "none",
      "installMaven": true
    },
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/vscode/.config/gh,type=bind",
    "source=maven-repo,target=/home/vscode/.m2,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "updateRemoteUserUID": true,
  "forwardPorts": [8080]
}
````

### Explicando cada campo

**`image`** — imagem base. Precisa bater com a versão do projeto (`java:17`, `java:21`, `python:3.12`, `typescript-node:22`, `ruby:3.3`, `rust:1`).

**`features`** — pedaços de instalação reaproveitáveis. Rodam no build, em ordem.

- `java:1` com `"version": "none"` → **não** instala outro JDK (a imagem já tem um); serve só para trazer o Maven via `installMaven`.
- `github-cli:1` → instala o `gh`. Substitui o MCP do GitHub; ver Etapa C4.7.
- `node:1` → **obrigatória**. A feature do Claude Code precisa de Node + npm. Sem ela, ela tenta instalar Node sozinha via apt, e o pacote do Debian vem **sem npm**, quebrando o build com `ERROR: Node.js and npm are required but could not be installed!`
- `claude-code:1` → instala o Claude Code.

**`mounts`** — o que sobrevive a rebuilds. Dois tipos:

- `type=bind` em `${localEnv:HOME}/.claude` → aponta para a pasta `~/.claude` **da tua máquina**. Com isso você ganha quatro coisas: o login persiste, o `CLAUDE.md` de usuário (preferências pessoais) é carregado, a config de MCP de escopo `user` vale em todos os projetos (ver Etapa C4.6), e tudo isso é igual em todos eles. `${localEnv:HOME}` é resolvido pela CLI para o teu home no host.
- `type=bind` em `${localEnv:HOME}/.config/gh` → mesma lógica para o login do GitHub CLI. Sem ele, você refaz `gh auth login` a cada rebuild.
- `type=volume` em `maven-repo` → cache de dependências gerenciado pelo Docker. Sem ele, o Maven rebaixa o Spring Boot inteiro a cada rebuild.

**Pré-requisito dos binds:** as pastas precisam existir no host antes de subir. Se não existirem, o Docker cria um diretório vazio (ou, no caso de um arquivo, cria um diretório com o nome do arquivo) e a ferramenta quebra.

````bash
mkdir -p ~/.claude ~/.config/gh
touch ~/.claude/CLAUDE.md
````

**Alternativa, se preferir isolar:** use `type=volume` com um nome fixo em vez do bind. O login persiste, mas fica dentro do Docker — o `CLAUDE.md` de usuário do host **não** é carregado, e a config de MCP não é compartilhada entre projetos.

````json
"source=claude-code-config,target=/home/vscode/.claude,type=volume"
````

**`containerEnv`** — variáveis de ambiente dentro do container. `CLAUDE_CONFIG_DIR` diz ao Claude onde ler e gravar toda a configuração, para coincidir com o mount. Isso inclui o `.claude.json`, que é onde ficam os servidores MCP — detalhe importante, explicado na Etapa C4.6.

**`remoteUser`** — o usuário dentro do container. **Muda conforme a imagem:**

| Imagem | Usuário | Caminho do home |
|---|---|---|
| `typescript-node` | `node` | `/home/node` |
| `java`, `python`, `ruby`, `rust` | `vscode` | `/home/vscode` |

Se errar aqui, os mounts vão para um caminho que não existe e o login não persiste.

**`updateRemoteUserUID`** — remapeia o uid do usuário do container para o **teu** uid do host. Existe exatamente para resolver o problema clássico de bind mount: o usuário do container tem uid 1000, e se o teu uid no host for diferente, os arquivos criados dentro do container saem com dono errado e você toma `Permission denied` no `~/.claude`.

Em Linux, com usuário não-root, o padrão já é `true` — está explícito aqui só para deixar a intenção clara. Se você tiver problema de permissão no `~/.claude`, confira `id -u` no host **antes** de assumir que precisa abrir mão do bind: na maioria dos casos o remapeamento já cobre.

**`forwardPorts`** — portas que ficam acessíveis do host. Sem isso, um servidor rodando no container não abre no teu navegador.

---

## Etapa C3 — COM banco (compose do time)

Aqui são **dois** arquivos.

### Arquivo 1

````bash
mkdir -p .devcontainer
nano .devcontainer/docker-compose.dev.yml
````

````yaml
services:
  db:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    image: mcr.microsoft.com/devcontainers/java:17
    volumes:
      - .:/workspaces/nome-do-projeto
    command: sleep infinity
    depends_on:
      db:
        condition: service_healthy
````

**Explicando:**

- `app` é um serviço **novo**, acrescentado ao compose do time. O `docker-compose.yml` original não é modificado — a CLI mescla os dois em memória.
- **`.` e não `..`** — este é o erro mais fácil de cometer. O Docker Compose resolve caminhos relativos contra a **raiz do projeto**, não contra a pasta onde o arquivo está. Com `..`, você monta a pasta *acima* do projeto, ou seja, todos os teus outros repositórios dentro do container.
- `command: sleep infinity` mantém o container vivo. Sem isso ele iniciaria e morreria imediatamente, já que a imagem base não tem processo de longa duração.
- **`healthcheck` + `condition: service_healthy`** — `depends_on` sozinho garante apenas a **ordem de start**, não que o banco esteja aceitando conexão. O Postgres leva alguns segundos entre "container subiu" e "pronto para conectar". Como o `app` só faz `sleep infinity`, você não percebe na hora — o sintoma aparece depois, quando o primeiro `mvn spring-boot:run` logo após o `devcontainer up` toma `connection refused` e você perde tempo procurando erro na configuração do datasource, que está certa. O `healthcheck` roda `pg_isready` dentro do próprio container do banco e só libera o `app` quando ele responde.
- Se o banco não for Postgres, troque o teste: MySQL usa `mysqladmin ping -h localhost`; Redis usa `redis-cli ping`.
- O bloco `db:` aqui **acrescenta** o healthcheck ao serviço que já existe no compose do time, sem redefinir imagem, portas ou variáveis. É merge, não substituição.

**Como verificar se o mount está certo:** no início da saída do `devcontainer up`, procure:

````
source: /home/mint/Documentos/Projetos/nome-do-projeto   ← certo
source: /home/mint/Documentos/Projetos                   ← ERRADO, pare
````

### Arquivo 2

````bash
nano .devcontainer/devcontainer.json
````

````json
{
  "name": "nome-do-projeto",
  "dockerComposeFile": [
    "../docker-compose.yml",
    "docker-compose.dev.yml"
  ],
  "service": "app",
  "workspaceFolder": "/workspaces/nome-do-projeto",
  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "none",
      "installMaven": true
    },
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/vscode/.config/gh,type=bind",
    "source=maven-repo,target=/home/vscode/.m2,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude",
    "SPRING_DATASOURCE_URL": "jdbc:postgresql://db:5432/postgresdb"
  },
  "remoteUser": "vscode",
  "updateRemoteUserUID": true,
  "forwardPorts": [8080]
}
````

**Campos novos em relação ao C2:**

**`dockerComposeFile`** — lista, em ordem. O primeiro é o do time (caminho relativo à pasta `.devcontainer/`, por isso `../`), o segundo é o teu. O segundo sobrescreve o primeiro em caso de conflito.

**`service`** — qual serviço vira o dev container. Tem que ser `app`, o que você criou no arquivo 1.

**`workspaceFolder`** — onde o Claude vai abrir. Precisa ser **exatamente** o `target` do volume no arquivo 1.

**`SPRING_DATASOURCE_URL`** — o pulo do gato. O `application.properties` do time aponta para `localhost:5432`, que funciona quando a app roda na máquina. Dentro do container, `localhost` é o próprio container — o banco está em outro. O host correto passa a ser o nome do serviço (`db`).

Como o `application.properties` é versionado e compartilhado, **não dá para editá-lo** — você quebraria o ambiente de todo mundo. A variável de ambiente resolve porque o Spring Boot dá precedência a env vars sobre o `.properties`. O time não vê diferença nenhuma.

> Equivalente em outras stacks: `DATABASE_URL` (Node/Prisma, Django), `DB_HOST` (Laravel).

---

## Etapa C4 — Visibilidade para a equipe

**Esta é a decisão que você toma antes de subir.**

### Opção 1 — Só para você (invisível ao time)

````bash
echo ".devcontainer/" >> .git/info/exclude
git status --short
````

O `git status` não pode listar `.devcontainer`.

**Como funciona:** `.git/info/exclude` tem a mesma sintaxe do `.gitignore`, mas vive dentro de `.git/` — ou seja, é local à tua cópia e **nunca vai para o repositório**. Ninguém no time vê nem o arquivo, nem a regra que o ignora.

**Por que não usar `.gitignore`:** o `.gitignore` é versionado. Se você colocasse `.devcontainer/` lá, o time veria a regra no próximo pull e saberia exatamente o que você está fazendo — além de impedir que *eles* commitem um devcontainer no futuro.

**Vale para qualquer branch.** O `.git/info/exclude` é por repositório, não por branch. Trocar de branch não afeta.

**Não sobrevive a clone novo.** Como o arquivo vive dentro de `.git/`, ele morre junto com a pasta. Se você reclonar o repositório, formatar a máquina ou trabalhar de outro computador, a regra some — o `.devcontainer/` volta a aparecer no `git status`, e um `git add .` distraído commita o teu ambiente pessoal no repo do time. Se for usar esta opção, deixe um lembrete no teu `~/.claude/CLAUDE.md` global para recriar a regra a cada clone.

**Risco de colisão:** se o time criar um `.devcontainer/` depois, o arquivo deles vem no pull e pode conflitar com o teu silenciosamente. Se isso for provável, use um nome diferente:

````bash
mkdir .devcontainer-local
echo ".devcontainer-local/" >> .git/info/exclude
devcontainer up --workspace-folder . --config .devcontainer-local/devcontainer.json
````

### Opção 2 — Compartilhado com o time (versionado)

Não faz nada — só commita:

````bash
git add .devcontainer/
git commit -m "chore: adiciona devcontainer para ambiente de desenvolvimento"
````

**Vantagens:** todo mundo com a mesma versão de Java, Maven e banco. Acaba o "na minha máquina funciona". Quem usa VS Code ou Cursor só aperta "Reopen in Container".

**Antes de commitar, tire o que é pessoal:**
- a feature `claude-code` (nem todo mundo usa Claude Code)
- o mount de `~/.claude`
- o `CLAUDE_CONFIG_DIR`

A feature `github-cli` pode ficar — é útil para qualquer pessoa.

Sobra um devcontainer genérico e útil para o time. Você mantém a tua parte à parte, com a Opção 3.

### Opção 3 — Híbrido (base compartilhada + tua camada) — **recomendada**

O time commita `.devcontainer/` sem Claude Code. Você acrescenta um arquivo local:

````bash
nano .devcontainer/devcontainer.local.json
echo "devcontainer.local.json" >> .git/info/exclude
````

E sobe apontando para ele:

````bash
devcontainer up --workspace-folder . --config .devcontainer/devcontainer.local.json
````

**Por que esta e não a Opção 1:** a Opção 1 te coloca sozinho num ambiente que ninguém mais reproduz. Em projeto de grupo, na primeira vez que algo compilar para você e não para eles (ou o contrário), você gasta tempo provando que não é o teu setup — e não tem como provar, porque de fato ninguém mais roda o que você roda. A Opção 3 entrega o mesmo isolamento, custa os mesmos dois arquivos, e ainda dá ao time o benefício do ambiente padronizado.

Se a leitura do grupo for que qualquer proposta de infraestrutura vira discussão longa, a Opção 1 é defensável **enquanto você valida** — só não deixe virar permanente.

---

## Etapa C4.5 — Onde ficam as instruções (`CLAUDE.md`)

O Claude Code lê instruções de dois lugares, e eles se somam:

| Arquivo | Escopo | Chega ao container? |
|---|---|---|
| `~/.claude/CLAUDE.md` (host) | tuas preferências, todos os projetos | **só com o bind da Etapa C2** |
| `CLAUDE.md` na raiz do repo | este projeto | sempre — está na pasta montada |

**O de usuário** é onde vão preferências de estilo: como você quer que ele responda, o que evitar, convenções tuas. Com o mount `type=bind`, ele é carregado automaticamente. Sem o bind (usando volume), ele não chega — você teria que recriar o arquivo dentro do container e manter duas cópias divergindo.

**O de projeto** vale para quem trabalhar naquele repositório. Aqui aparece a mesma decisão de visibilidade da Etapa C4:

**Se for pessoal** — esconda como o devcontainer:

````bash
nano CLAUDE.md
echo "CLAUDE.md" >> .git/info/exclude
git status --short
````

**Se for para o time** — commite. Convenções de código, arquitetura, como rodar os testes, o que não mexer: isso ajuda qualquer pessoa usando qualquer assistente, e costuma ser mais valioso compartilhado do que escondido.

````bash
git add CLAUDE.md
git commit -m "docs: adiciona CLAUDE.md com convenções do projeto"
````

**Divisão sugerida:** preferências de estilo pessoal no global (`~/.claude/CLAUDE.md`); fatos sobre o projeto no `CLAUDE.md` do repo, versionado.

> Como escrever o do projeto na prática, com modelo pronto: **Parte F**, no final deste guia.

---

## Etapa C4.6 — Servidores MCP dentro do container

Esta seção existe porque o comportamento aqui não é óbvio e gera meia hora perdida.

### O ponto que confunde

`CLAUDE.md` e a configuração de MCP **não moram no mesmo lugar**:

- `~/.claude/CLAUDE.md` → dentro do **diretório** `~/.claude/`
- `~/.claude.json` → um **arquivo solto** na raiz da home, irmão do diretório, não filho dele

Por padrão, montar `~/.claude` traria o `CLAUDE.md` e deixaria a config de MCP para trás. Mas o `CLAUDE_CONFIG_DIR` da Etapa C2 muda isso: com ele apontando para `/home/vscode/.claude`, o Claude Code passa a ler o `.claude.json` **de dentro** desse diretório — ou seja, `~/.claude/.claude.json` no host, que está no bind.

Consequência prática: **o `~/.claude.json` da raiz da tua home continua irrelevante para o container.** Instalar MCP no host, por fora, não adianta. Duas razões:

1. O `CLAUDE_CONFIG_DIR` redireciona a leitura para outro caminho.
2. Mesmo sem ele, o escopo `local` é indexado pelo **caminho absoluto do projeto** — `/home/manoel/projeto` no host, `/workspaces/projeto` no container. As chaves não batem, e você recebe `No MCP servers configured`.

### O que fazer

**MCP pessoal, de uso geral** (Sentry, docs) → escopo `user`, rodando **de dentro** do container:

````bash
dcsh
claude mcp add --scope user --transport http sentry https://mcp.sentry.dev/mcp
````

Grava em `~/.claude/.claude.json` no host, pelo bind. Vale em todos os devcontainers que usem o mesmo mount. Configura uma vez, funciona em todo lugar.

> **GitHub não entra aqui — veja a Etapa C4.7.** O endpoint MCP oficial exige assinatura do GitHub Copilot, e a solução por CLI é melhor de qualquer forma.

**MCP do projeto** (Sentry daquele serviço, Supabase daquele banco) → `.mcp.json` na raiz do repositório:

````json
{
  "mcpServers": {
    "sentry": { "type": "http", "url": "https://mcp.sentry.dev/mcp" }
  }
}
````

Como fica na pasta montada, funciona no host e no container sem depender de home nem de caminho absoluto. E é versionável, se o time quiser.

### Alternativa: configurar pelo host

Se você também tem Claude Code instalado na máquina, dá para rodar o `add` no host e o resultado valer no container — desde que o host grave no mesmo arquivo:

````bash
CLAUDE_CONFIG_DIR=~/.claude claude mcp add --scope user \
  --transport http sentry https://mcp.sentry.dev/mcp
````

Para valer sempre, sem prefixar o comando:

````bash
echo 'export CLAUDE_CONFIG_DIR="$HOME/.claude"' >> ~/.bashrc
source ~/.bashrc
cp ~/.claude.json ~/.claude/.claude.json   # leva o que já existia
````

**O ganho real disso é o OAuth.** O problema de "o navegador não abre dentro do container" desaparece: você autentica no host, com navegador normal, e o token fica em `~/.claude/`, que o container lê. Autentica uma vez, funciona nos dois.

**Não vale para `stdio`.** O que fica gravado é o comando (`npx -y ...`), e ele é executado **onde o Claude está rodando**. Registrar no host não instala nada no container.

### Três armadilhas específicas de container

- **Servidores `stdio` rodam dentro do container.** `npx @playwright/mcp` precisa de Node **no container** e dos binários do Chromium, que não vêm na imagem. E Playwright abre janela de browser: sem display, tem que ser headless. Não é um `add` e pronto.
- **Servidores via `docker run`** exigem docker-in-docker ou socket montado. Prefira a variante HTTP.
- **OAuth não abre navegador no container.** Rode `/mcp`, copie a URL que aparece no terminal e cole no navegador do host manualmente — ou configure pelo host, como acima.

Por isso, em devcontainer: **prefira MCP HTTP com token estático (`--header`) a `stdio` e a OAuth.** Menos coisa para quebrar no rebuild.

### Custo de contexto

Cada servidor conectado carrega os nomes das ferramentas e as instruções do servidor em **toda** sessão, ocupando context window. Instalar seis MCPs "por garantia" custa contexto em todo prompt.

**Atenção aos conectores do claude.ai.** Os conectores ativados em claude.ai/customize/connectors carregam automaticamente no CLI quando você está logado com a mesma conta — aparecem no `claude mcp list` com o prefixo `claude.ai`. Você não os instalou nessa máquina, e `claude mcp remove` **não funciona neles**, porque não estão no teu `.claude.json`. Com uma dúzia deles ligados, você paga dezenas de milhares de tokens de contexto por sessão sem perceber, e aumenta a chance do agente escolher a ferramenta errada entre opções parecidas.

Desative na origem, em claude.ai/customize/connectors. Conectores de produtividade (Drive, Gmail, Notion, Microsoft 365, Zapier) não têm função dentro de uma sessão de código.

````bash
claude mcp list          # status de cada um
claude mcp remove <nome> # só funciona nos que você instalou
````

---

## Etapa C4.7 — GitHub: use `gh`, não MCP

### O que não funciona

O servidor MCP oficial do GitHub fica em `https://api.githubcopilot.com/mcp`. O nome entrega: é o endpoint do **GitHub Copilot**, produto pago. Sem assinatura ativa, ele recusa a conexão e o `claude mcp list` mostra `✘ Failed to connect` — independente do PAT ter todas as permissões do mundo.

Diagnóstico em um comando:

````bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer SEU_TOKEN" \
  https://api.githubcopilot.com/mcp
````

`401` ou `403` → é falta de Copilot. Não insista, não tem contorno por token.

**Duas confusões comuns nesse erro:**

- *"É porque estou no container?"* Não. MCP HTTP é uma chamada de rede para um servidor remoto — funciona igual no host e no container.
- *"Preciso instalar git no container?"* Não. O servidor MCP roda do outro lado da internet; ele não executa binário nenhum na tua máquina.

### O que funciona melhor

O Claude Code executa comandos no terminal. Basta o **GitHub CLI** estar disponível — issues, PRs, reviews e checks, tudo pela ferramenta que ele já tem.

Três vantagens sobre o MCP:

1. **Não custa context window.** MCP carrega descrições de ferramentas em toda sessão; um binário no PATH não carrega nada.
2. **Serve para você também.** O `gh` é útil na mão, o MCP não.
3. **Não quebra no rebuild.** É uma feature do devcontainer, reinstalada no build.

### Configuração

Já está nos exemplos das Etapas C2 e C3 — a feature `github-cli:1` e o bind de `~/.config/gh`. O bind é o que faz o login sobreviver ao rebuild; sem ele, você refaz `gh auth login` toda vez que recriar o container.

Crie a pasta no host **antes** de subir — mesma regra do `~/.claude`, senão o Docker cria um diretório vazio:

````bash
mkdir -p ~/.config/gh
devcontainer up --workspace-folder . --remove-existing-container
dcsh
gh auth login
gh repo view    # confirma
````

### O que documentar no `CLAUDE.md`

Quase nada sobre o `gh` em si — o Claude descobre o comando sozinho, e explicar que ele existe é gastar contexto com o óbvio.

O que ele **não** adivinha são as regras do teu repositório. É aí que o `CLAUDE.md` vale: como guarda-corpo, não como manual. Abrir PR contra `main` num projeto de grupo é exatamente o tipo de erro que ele comete sozinho e que dá trabalho para desfazer. Ver Parte F.

---

## Etapa C5 — Derrubar containers conflitantes

````bash
docker compose down
````

**Por quê:** se o `docker-compose.yml` do time define `container_name` fixo (ex: `my-journey-db`), esse nome é único no Docker inteiro. Se o banco já estiver de pé por um `docker compose up` normal, a CLI não consegue criar o dela e falha.

Alternativa, se quiser manter os dois ambientes: sobrescreva o nome no teu `docker-compose.dev.yml`:

````yaml
services:
  db:
    container_name: nome-do-projeto-db-dev
````

---

## Etapa C6 — Subir

````bash
devcontainer up --workspace-folder .
````

**O que acontece:** baixa a imagem base, roda as features em ordem, cria os volumes, sobe `db` e espera ele ficar saudável, depois sobe `app` na mesma rede.

Primeira vez: vários minutos. Depois: segundos.

Terminou bem quando aparece:
````json
{"outcome":"success","containerId":"...","remoteUser":"vscode","remoteWorkspaceFolder":"/workspaces/nome-do-projeto"}
````

---

## Etapa C7 — Corrigir permissão dos volumes

````bash
dcsh
````

Dentro do container:

````bash
sudo chown -R vscode:vscode /home/vscode/.m2
````

**Por quê:** volume nomeado criado vazio nasce pertencendo ao root. Como o container roda como `vscode`, o Maven não consegue escrever e falha com `Could not create local repository`. As imagens devcontainer dão sudo sem senha, então o chown resolve de dentro.

**Só precisa fazer uma vez por volume.** Depois disso ele fica com o dono certo para sempre.

Mesma lógica para outros caches: `.gradle`, `.cargo`, `bundle`, e o volume de `node_modules` (ver a seção de Node na Parte C-BIS).

---
## Etapa C7.5 — Auto-update do Claude Code sem permissão

### O sintoma

Ao abrir uma sessão, aparece:

```
✘ Auto-update failed: no write permission to npm prefix · Run claude doctor
```

Nada quebra. A versão instalada funciona normalmente, você trabalha o dia
inteiro sem notar diferença — mas o Claude Code **nunca se atualiza sozinho**.
O container vai ficando para trás versão após versão, em silêncio.

### A causa

As features do devcontainer rodam **como root** durante o build. A feature
`ghcr.io/anthropics/devcontainer-features/claude-code:1` instala o pacote em
`$(npm config get prefix)/lib/node_modules/@anthropic-ai/`, e essa pasta fica
com dono `root` e permissão `drwxr-sr-x`.

O container, porém, roda com `"remoteUser": "vscode"`. Quando o Claude tenta
gravar a versão nova ali, não tem permissão de escrita — e desiste.

É o mesmo padrão da Etapa C7: algo criado por root no build, usado por um
usuário não-root em tempo de execução.

### A correção

Manter `root` como dono e dar escrita ao **grupo** `nvm` — o usuário `vscode`
já pertence a ele — mais o bit **setgid** nos diretórios.

No `devcontainer.json`:

```json
"postCreateCommand": "D=$(npm config get prefix)/lib/node_modules/@anthropic-ai; if [ -d \"$D\" ]; then sudo chmod -R g+w \"$D\"; sudo find \"$D\" -type d -exec chmod g+s {} +; fi"
```

O `postCreateCommand` é o lugar certo porque roda **depois** das features. Um
comando executado antes não encontraria a pasta — ela ainda não existe. E, por
estar no `devcontainer.json`, a correção é reaplicada a cada rebuild, sem você
precisar lembrar.

O `if [ -d ... ]` evita que o comando falhe em imagem onde a feature não foi
instalada.

### Por que o setgid importa

Esta é a parte que costuma ser esquecida, e sem ela a correção dura uma
atualização só.

O `chmod -R g+w` resolve o **agora**: os arquivos que existem passam a ser
graváveis pelo grupo `nvm`. Mas o update não só sobrescreve arquivos — ele
**cria** arquivos e pastas novos. E, por padrão, um arquivo criado pelo usuário
`vscode` nasce no **grupo pessoal dele**, não em `nvm`.

Resultado: a atualização seguinte encontra arquivos fora do grupo que tem
permissão de escrita, e o erro volta.

O bit setgid (`chmod g+s`) em um diretório muda essa regra: tudo que for criado
lá dentro **herda o grupo do diretório** em vez do grupo do usuário. Com ele, os
arquivos do update nascem já em `nvm`, com escrita de grupo, e o ciclo se
sustenta sozinho.

Por isso o `find -type d`: setgid só faz sentido em diretório, e precisa valer
para toda a árvore, não só para a raiz.

### Duas "soluções" que aparecem em fórum e que você deve evitar

**`chmod 777`** — resolve o sintoma abrindo escrita para qualquer processo do
container, inclusive os que você não controla. Você trocou um aviso chato por
uma permissão que não tem como justificar depois. O `g+w` dá exatamente o acesso
necessário, para exatamente quem precisa dele.

**Rodar o container como root** (remover o `remoteUser` ou apontá-lo para
`root`) — desfaz boa parte do motivo de estar usando devcontainer. Todo arquivo
que o agente criar no projeto sai com dono root no teu host, pelo bind mount, e
você passa a precisar de `sudo` para editar o próprio código. É um problema bem
maior que o auto-update parado.

### Se você já tem um `postCreateCommand`

Só existe um por arquivo. Encadeie com `&&`:

```json
"postCreateCommand": "npm ci && D=$(npm config get prefix)/lib/node_modules/@anthropic-ai; if [ -d \"$D\" ]; then sudo chmod -R g+w \"$D\"; sudo find \"$D\" -type d -exec chmod g+s {} +; fi"
```

### Verificar

Recrie o container e confira as permissões:

```bash
devcontainer up --workspace-folder . --remove-existing-container
dcsh
ls -ld $(npm config get prefix)/lib/node_modules/@anthropic-ai
```

O que você quer ver: dono `root`, grupo `nvm`, e `rws` no bloco do grupo — o
`s` no lugar do `x` é o setgid ativo.

---
## Etapa C8 — Validar

Ainda dentro do `dcsh`:

````bash
claude --version                  # Claude Code instalado
gh --version                      # GitHub CLI presente
java -version                     # versão certa
mvn -v | head -1                  # Maven presente
echo $SPRING_DATASOURCE_URL       # aponta para o serviço, não localhost
ls                                # só ESTE projeto (pom.xml, src, ...)
mvn -q compile                    # o teste definitivo
````

O `mvn -q compile` valida tudo de uma vez: JDK, Maven, permissão do `.m2` e acesso ao código. Com `-q`, **saída vazia significa sucesso**.

````bash
exit
````

---

## Etapa C9 — Usar

````bash
dc
````

Primeira execução pede login: ele imprime uma URL, você abre no navegador do host, autoriza e cola o código de volta.

Do dia a dia em diante: `cd` no projeto, `dc`.

---

# PARTE C-BIS — Configurações completas por linguagem

Os exemplos da Parte C usam Java. Abaixo, o `devcontainer.json` pronto para as
outras stacks, na versão **sem banco** (Etapa C2).

Para a versão **com banco** (Etapa C3), troque `"image"` por
`"dockerComposeFile"`, `"service"` e `"workspaceFolder"`, mova a imagem para o
`docker-compose.dev.yml`, acrescente o `healthcheck` no serviço do banco, e
inclua a variável de conexão em `containerEnv` — o resto é idêntico.

Em **todas** as linguagens, a feature `node:1` vem antes da `claude-code:1`.

---

## Node / TypeScript

Usuário: **`node`** (atenção — é o único que não é `vscode`).

````json
{
  "name": "nome-do-projeto",
  "image": "mcr.microsoft.com/devcontainers/typescript-node:22",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/node/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/node/.config/gh,type=bind",
    "source=${localWorkspaceFolderBasename}-node-modules,target=/workspaces/${localWorkspaceFolderBasename}/node_modules,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/node/.claude"
  },
  "remoteUser": "node",
  "updateRemoteUserUID": true,
  "postCreateCommand": "npm ci",
  "forwardPorts": [3000]
}
````

- A feature `node:1` é dispensável aqui — a imagem já traz Node e npm.
- Todos os caminhos usam `/home/node`, não `/home/vscode`. Se errar, o mount vai
  para um caminho inexistente e o login não persiste.
- **O volume sobre `node_modules` não é opcional se você também roda o projeto no
  host.** É o mesmo problema do `.venv` em Python: pacotes com binário nativo
  (`bcrypt`, `sharp`, `esbuild`, `better-sqlite3`) são compilados para o ambiente
  onde o install rodou. Sem o volume, o `npm ci` do container sobrescreve o
  binário do host e vice-versa, e você fica alternando entre
  `invalid ELF header` nos dois lados. O volume dá a cada ambiente o seu
  `node_modules`, na mesma pasta aparente.
- Esse volume **vai precisar do chown da Etapa C7**:
  `sudo chown -R node:node /workspaces/nome-do-projeto/node_modules`
- Se você **só** roda no container, pode dispensar o volume — mas aí o `npm ci`
  refaz o install a cada rebuild.
- Trocar a versão: `typescript-node:20`, `:22`, `:24`.
- Se o projeto usa pnpm ou yarn, troque `npm ci` por `pnpm install` ou
  `yarn install`.
- Variável de conexão típica com banco: `DATABASE_URL`.

---

## Python

Usuário: **`vscode`**.

````json
{
  "name": "nome-do-projeto",
  "image": "mcr.microsoft.com/devcontainers/python:3.12",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/vscode/.config/gh,type=bind"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "updateRemoteUserUID": true,
  "postCreateCommand": "python -m venv .venv && .venv/bin/pip install -r requirements.txt",
  "forwardPorts": [8000]
}
````

- Trocar a versão: `python:3.11`, `:3.12`, `:3.13`.
- Com Poetry: `"postCreateCommand": "pipx install poetry && poetry install"`.
- Sem `requirements.txt`: use só `python -m venv .venv`.
- Adicione `.venv/` ao `.gitignore`.
- O `.venv` fica dentro do projeto, que é bind mount — então não precisa de
  volume de cache. Mas se você também roda Python no host, os dois vão brigar
  pela mesma pasta (binários compilados para ambientes diferentes). Nesse caso,
  use um caminho separado: `python -m venv /home/vscode/.venv-container`.
  É o mesmo problema descrito na seção de Node.
- Variável de conexão típica: `DATABASE_URL`.

---

## Ruby

Usuário: **`vscode`**.

````json
{
  "name": "nome-do-projeto",
  "image": "mcr.microsoft.com/devcontainers/ruby:3.3",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/vscode/.config/gh,type=bind",
    "source=bundle-cache,target=/usr/local/bundle,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "updateRemoteUserUID": true,
  "postCreateCommand": "bundle install",
  "forwardPorts": [3000]
}
````

- Trocar a versão: `ruby:3.2`, `:3.3`, `:3.4`.
- O volume `bundle-cache` é o equivalente ao `maven-repo`: sem ele, o
  `bundle install` refaz tudo a cada rebuild.
- **Vai precisar do chown da Etapa C7**, com o caminho do bundle:
  `sudo chown -R vscode:vscode /usr/local/bundle`
- Sem `Gemfile`, remova o `postCreateCommand`.
- Em Rails, a variável de conexão costuma vir do `config/database.yml`; use
  `DATABASE_URL` para sobrescrever sem editar o arquivo versionado.

---

## Rust

Usuário: **`vscode`**.

````json
{
  "name": "nome-do-projeto",
  "image": "mcr.microsoft.com/devcontainers/rust:1",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {},
    "ghcr.io/devcontainers/features/node:1": {},
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    "source=${localEnv:HOME}/.claude,target=/home/vscode/.claude,type=bind",
    "source=${localEnv:HOME}/.config/gh,target=/home/vscode/.config/gh,type=bind",
    "source=cargo-registry,target=/usr/local/cargo/registry,type=volume"
  ],
  "containerEnv": {
    "CLAUDE_CONFIG_DIR": "/home/vscode/.claude"
  },
  "remoteUser": "vscode",
  "updateRemoteUserUID": true,
  "forwardPorts": [8080]
}
````

- Vem com `rustup` completo: `rustup target add`, `rustup component add clippy`,
  troca de canal.
- **Vai precisar do chown da Etapa C7:**
  `sudo chown -R vscode:vscode /usr/local/cargo/registry`
- O `target/` de build fica dentro do projeto e pode ficar pesado; considere um
  volume para ele também se o rebuild incomodar.

---

## Java (referência — igual à Etapa C2)

Usuário: **`vscode`**. Ver Etapa C2 para o arquivo completo e a explicação de
cada campo.

- `"version": "none"` na feature `java:1` evita instalar um segundo JDK.
- Volume `maven-repo` para o cache de dependências.
- Com Gradle: use `./gradlew` do projeto; se não houver wrapper, acrescente
  `"installGradle": true` na feature.

---

## Projeto poliglota

Se o repositório tem, por exemplo, backend Java e frontend Node, escolha a
imagem da linguagem **principal** e acrescente a outra por feature:

````json
"image": "mcr.microsoft.com/devcontainers/java:17",
"features": {
  "ghcr.io/devcontainers/features/java:1": { "version": "none", "installMaven": true },
  "ghcr.io/devcontainers/features/node:1": { "version": "22" },
  "ghcr.io/devcontainers/features/github-cli:1": {},
  "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
}
````

Features existem para Python, Ruby, Go, .NET e outras — o padrão do nome é
`ghcr.io/devcontainers/features/<linguagem>:1`.

**Contraponto:** empilhar quatro linguagens numa imagem só porque "pode" deixa o
build lento e a superfície maior sem retorno. Adicione a segunda linguagem
quando o projeto realmente precisar dela, não por precaução.

---

# PARTE D — Referência rápida

## Comandos

| Situação | Comando |
|---|---|
| Abrir o Claude | `dc` |
| Shell no container | `dcsh` |
| Depois de mudar o `devcontainer.json` | `devcontainer up --workspace-folder . --remove-existing-container` |
| Ver os volumes | `docker volume ls` |
| Ver containers de pé | `docker ps` |
| Listar servidores MCP | `claude mcp list` |
| Login do GitHub CLI | `gh auth login` (dentro do container) |

## Imagens por linguagem

| Linguagem | Imagem | `remoteUser` |
|---|---|---|
| Node/TS | `mcr.microsoft.com/devcontainers/typescript-node:22` | `node` |
| Java | `mcr.microsoft.com/devcontainers/java:17` (ou `:21`) | `vscode` |
| Python | `mcr.microsoft.com/devcontainers/python:3.12` | `vscode` |
| Ruby | `mcr.microsoft.com/devcontainers/ruby:3.3` | `vscode` |
| Rust | `mcr.microsoft.com/devcontainers/rust:1` | `vscode` |

Em **todas**, inclua a feature `node:1` antes da `claude-code:1`.

## Erros e causas

| Erro | Causa | Solução |
|---|---|---|
| `Node.js and npm are required but could not be installed!` | falta a feature de Node | adicionar `"ghcr.io/devcontainers/features/node:1": {}` |
| Monta a pasta errada | `..` no `docker-compose.dev.yml` | trocar por `.` |
| `Could not create local repository` | volume pertence ao root | `sudo chown -R vscode:vscode /home/vscode/.m2` |
| `container_name` já em uso | banco já rodando | `docker compose down` |
| Pede login toda vez | mount do `.claude` no caminho errado | conferir `remoteUser` vs. caminho do home |
| App não conecta no banco | aponta para `localhost` | definir a env var do datasource com o nome do serviço |
| `connection refused` no primeiro run | banco subiu mas ainda não aceitava conexão | `healthcheck` + `condition: service_healthy` |
| `git commit` falha | falta identidade do git | `git config --global user.name/user.email` |
| `nc: command not found` | ferramenta não existe na imagem | não é erro de rede; testar com o build real |
| `CLAUDE.md` de usuário ignorado | mount é `type=volume`, não bind | trocar por `source=${localEnv:HOME}/.claude,...,type=bind` |
| `~/.claude` vazio dentro do container | a pasta não existia no host | `mkdir -p ~/.claude` antes de subir |
| Permission denied no `~/.claude` | uid do host ≠ 1000 e remapeamento desligado | conferir `updateRemoteUserUID: true` |
| `No MCP servers configured` | MCP instalado no host, não no container | `claude mcp add --scope user` de dentro do container |
| `invalid ELF header` em pacote nativo | `node_modules` compartilhado entre host e container | volume sobre `node_modules` |
| `✘ Failed to connect` no MCP do GitHub | endpoint exige assinatura do Copilot | usar `gh` por feature (Etapa C4.7) |
| `gh` pede login a cada rebuild | falta o bind de `~/.config/gh` | adicionar o mount e `mkdir -p ~/.config/gh` |
| Dezenas de MCPs na lista que não dá para remover | conectores do claude.ai carregando automaticamente | desativar em claude.ai/customize/connectors |

---

# PARTE E — Limitações e cuidados

**O container isola o filesystem, não a rede.** O agente ainda pode fazer `git push`, `curl` para qualquer endereço e usar as credenciais montadas. Se for usar `--dangerously-skip-permissions`, vale olhar o devcontainer de referência da Anthropic (`anthropics/claude-code/.devcontainer`), que traz um `init-firewall.sh` com política default-deny.

**Os binds incluem as tuas credenciais.** `~/.claude` guarda o token de autenticação da tua conta Anthropic; `~/.config/gh` guarda o do GitHub. Um agente rodando com `--dangerously-skip-permissions` dentro do container consegue lê-los — o isolamento de filesystem que você comprou com o devcontainer **não cobre isso**. É o preço consciente de ter login único e `CLAUDE.md` global; se o projeto executa código de terceiros ou dependências que você não auditou, reconsidere e use `type=volume`.

**Volumes com Compose ganham prefixo; binds não.** O que você declara como volume `claude-code-config` vira `nome-do-projeto_claude-code-config` — ou seja, cada projeto teria o seu, e você faria login separado em cada um. É por isso que o guia usa `type=bind` apontando para `~/.claude`: como é um caminho real do host, ele é literalmente o mesmo em todos os projetos. Um login só, `CLAUDE.md` de usuário e config de MCP junto.

**O outro preço do bind:** o container escreve direto no teu host. Uma mudança de formato de config feita pelo Claude do container vale para o host também. Se você quiser isolamento estrito por projeto, volte ao `type=volume` — abrindo mão do `CLAUDE.md` global, do MCP compartilhado e de logar uma vez só.

**Reboot não quebra nada.** Imagem, volumes, arquivos e a função `dc` são todos persistentes. Os containers param, mas `devcontainer up` os reinicia em segundos.

**Não rode `docker system prune -a`.** Ele remove imagens não utilizadas de todos os projetos, incluindo devcontainers de outros repositórios.

---

# PARTE F — Criando o `CLAUDE.md` do projeto

## F1 — Como os dois arquivos convivem

Não é um ou outro: os dois carregam juntos e se somam. Quando há conflito, o mais específico vence.

| Arquivo | Caminho dentro do container | Escopo |
|---|---|---|
| Global (seu) | `/home/vscode/.claude/CLAUDE.md` | todos os projetos |
| Do projeto | `/workspaces/nome-do-projeto/CLAUDE.md` | só este repositório |

O global chega ao container pelo `type=bind` da Etapa C2. O do projeto já está lá — é um arquivo do repo, dentro da pasta montada.

## F2 — O que vai em cada um

**Regra:** o global descreve **você**. O do projeto descreve **o repositório**.

| Global | Do projeto |
|---|---|
| tom e estilo de resposta | stack e versões |
| limites de segurança | arquitetura e convenções de organização |
| fluxo de trabalho pessoal | comandos para validar |
| preferências de código | armadilhas conhecidas do repo |

**Não duplique.** Se uma regra está nos dois, na primeira vez que você atualizar um e esquecer o outro, eles se contradizem — e aí o Claude segue o do projeto, que pode ser o desatualizado.

**Documente o que ele não adivinha.** Ferramentas disponíveis no PATH ele descobre sozinho; não gaste linhas explicando que `gh` ou `mvn` existem. O valor está nas regras do repositório — qual branch, o que nunca fazer, o que já deu errado antes.

## F3 — Criar

````bash
cd ~/caminho/do/projeto
nano CLAUDE.md
````

## F4 — Modelo (exemplo real: Java + Spring Modulith + Postgres)

````markdown
# my-journey-core

Projeto em grupo. API Spring Boot com arquitetura modular.

## Stack

- Java 17, Maven (sem wrapper — use `mvn`)
- Spring Boot 3.4.1, Spring Modulith 1.3.0
- PostgreSQL 16 via `docker-compose.yml`
- Spring Security + JJWT para autenticação
- springdoc-openapi para documentação
- Lombok

## Ambiente

O banco sobe pelo `docker-compose.yml` na raiz (serviço `db`, base `postgresdb`).
Dentro do devcontainer, o host do banco é `db`, não `localhost` — já configurado
via `SPRING_DATASOURCE_URL`.

## Como validar

Antes de considerar qualquer task pronta:

```bash
mvn -q compile     # compila (saída vazia = sucesso)
mvn test           # testes, incluindo os de modularidade do Modulith
```

## Arquitetura

Spring Modulith: cada módulo é um pacote de primeiro nível sob o pacote raiz.
Comunicação entre módulos acontece pela API pública do módulo ou por eventos —
nunca importando classe interna de outro módulo. Os testes de modularidade
falham se essa regra for violada; não desative esses testes para fazer o build
passar.

## Git e GitHub

O `gh` (GitHub CLI) está disponível e autenticado no container — use para
issues, PRs e checks em vez de pedir ao usuário.

Regras que não podem ser violadas:

- Branch de trabalho é `dev`. Nunca commite nem abra PR direto para `main`.
- PR sempre com base em `dev`: `gh pr create --base dev`.
- Nunca use `push --force` em branch compartilhada.
- Commits seguem Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`).
- Projeto em grupo: não reescreva histórico, não faça rebase de branch que
  já foi enviada.

## Armadilhas conhecidas

- **Flyway e `ddl-auto` convivem hoje.** O `pom.xml` traz `flyway-core`, mas o
  `application.properties` está com `spring.jpa.hibernate.ddl-auto=update`. Os
  dois gerenciam schema ao mesmo tempo, o que faz o schema divergir entre as
  máquinas do time. Não "resolva" isso sozinho — é decisão do grupo.
- **O serviço `db` não tem volume.** `docker compose down` apaga os dados.
- **Senha do banco versionada** em `application.properties`.
````

> Confira as regras de branch e commit contra o combinado do teu grupo antes de
> commitar. Instrução errada é pior que instrução ausente.

## F5 — Decidir a visibilidade

Mesma escolha da Etapa C4.

**Pessoal (invisível ao time):**

````bash
echo "CLAUDE.md" >> .git/info/exclude
git status --short
````

Lembre que essa regra não sobrevive a um clone novo.

**Compartilhado (recomendado quando o conteúdo é sobre o projeto):**

````bash
git add CLAUDE.md
git commit -m "docs: adiciona CLAUDE.md com convenções do projeto"
````

Um `CLAUDE.md` de projeto bem escrito ajuda qualquer pessoa usando qualquer
assistente — e serve como documentação de onboarding mesmo para quem não usa
nenhum. É bem mais fácil de propor ao grupo do que o devcontainer, porque não
exige que ninguém mude o próprio ambiente.

## F6 — `CLAUDE.md` em subpastas (opcional)

Também é possível ter um `CLAUDE.md` dentro de um subdiretório; ele carrega
quando o Claude toca arquivos daquela pasta. Em projeto Modulith, um por módulo
pode fazer sentido.

**Só use quando as regras realmente divergirem entre as pastas.** Se o conteúdo
for o mesmo, você criou manutenção sem retorno — e mais uma fonte de
contradição.

## F7 — Manter atualizado

O maior risco de um `CLAUDE.md` não é estar incompleto: é estar **errado**.
Instrução desatualizada é pior que instrução ausente, porque o agente segue com
confiança.

Revise quando:
- a versão de uma dependência principal mudar
- um comando de build ou teste mudar
- uma armadilha listada for resolvida (remova a entrada)
- uma convenção nova for combinada com o time
