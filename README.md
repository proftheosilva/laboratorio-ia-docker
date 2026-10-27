# Laboratório de IA Generativa com Docker

Template didático da Cucando TI para reunir aplicações independentes em Docker. O estado inicial executa **somente o n8n**, com persistência, pastas de dados e rede compartilhada.

O aluno desenvolve e testa cada aplicação em seu repositório. Depois, clona o projeto em `apps/`, restaura seu `.env`, cria ou adapta o Dockerfile do próprio app e acrescenta um serviço no `compose.yaml`. Não há instalação antecipada de bibliotecas de RAG, MCP ou agentes.

## 1. Estrutura inicial

```text
laboratorio-ia-docker/
├── compose.yaml                 # Somente n8n
├── README.md
├── .gitignore
├── .env.example                 # Orientação opcional; não é exigido ao iniciar
├── apps/
│   ├── .gitkeep
│   └── README.md
├── dados/documentos/
│   ├── raw/
│   ├── silver/
│   └── gold/
└── docs/PLANO.md
```

Os Dockerfiles e dependências mostrados neste README são **exemplos para criar dentro das aplicações quando elas forem incorporadas**. Esses arquivos não vêm pré-criados no template.

## 2. Iniciar no Windows 11

Instale Git e Docker Desktop com backend WSL 2. Deixe o mecanismo Docker iniciado. No PowerShell:

```powershell
git clone https://github.com/proftheosilva/laboratorio-ia-docker.git
cd laboratorio-ia-docker
docker compose up -d
docker compose ps
```

Abra [http://localhost:5678](http://localhost:5678) e crie a conta local do n8n. O primeiro início baixa a imagem. O Compose mantém n8n `2.44.5` como versão de referência do laboratório; avalie atualizações antes da aula.

Python, Node.js, npm e uv não precisam estar no Windows para executar os serviços em contêineres. O desenvolvimento fora do Docker pode exigir essas ferramentas.

O Compose inicial não exige copiar `.env.example`. Credenciais do n8n são configuradas na interface.

## 3. Clonar os projetos em apps

```powershell
cd apps
git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO_MCP.git mcp
git clone https://github.com/SEU_USUARIO/SEU_REPOSITORIO_RAG.git rag
cd ..
```

Substitua as URLs pelos repositórios reais. O último argumento de clone determina o nome da pasta. Cada app continua sendo um repositório independente.

Após restaurar as credenciais e preparar os arquivos Docker:

```text
apps/
├── mcp/
│   ├── .env                     # Local, restaurado pelo aluno
│   ├── .env.example             # Somente nomes e valores fictícios
│   ├── requirements.txt
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── server.py
│   └── server_http.py
└── rag/
    ├── .env
    ├── requirements.txt
    ├── Dockerfile
    ├── .dockerignore
    ├── backend/
    └── frontend/
        ├── package.json
        ├── package-lock.json
        ├── Dockerfile
        └── .dockerignore
```

Traga o `.env` privado do pendrive ou armazenamento privado para a raiz do app: `apps/mcp/.env`, `apps/rag/.env`, etc. Não copie `venv`, `.venv`, `node_modules` ou caches; o Docker instala os pacotes dentro da imagem.

O `env_file` disponibiliza variáveis ao iniciar o contêiner. Não instala dependências. O `.env` na raiz do laboratório tem outro propósito: substituir expressões como `${VARIAVEL}` no Compose. O arquivo inicial não usa essas expressões.

Cada repositório de app precisa de seu próprio `.gitignore`. As regras do laboratório não protegem commits feitos dentro de outro repositório Git.

## 4. Qual arquivo faz o quê?

| Arquivo | Função |
|---|---|
| compose.yaml | Organiza serviços, portas, variáveis, pastas e rede |
| Dockerfile do app | Constrói a imagem e define como iniciar o app |
| requirements.txt do app | Declara as dependências daquele projeto Python |
| package.json e lockfile | Declaram dependências e comandos Node.js |
| .dockerignore do app | Evita enviar segredos e ambientes locais ao build |

Imagem é o ambiente construído; contêiner é uma execução dessa imagem. Um projeto pode usar vários serviços, como backend Python e frontend Next.js.

O nome convencional é **Dockerfile**, sem extensão. Em Python, é **requirements.txt**. Se o projeto usar outro nome, adapte o COPY e o pip install para esse nome exato.

## 5. Incorporar um MCP Python

Crie `apps/mcp/Dockerfile`, adaptando a versão Python ao projeto:

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

# Instala as dependências do próprio MCP.
COPY requirements.txt .
RUN python -m pip install --no-cache-dir -r requirements.txt

COPY . .
CMD ["python", "server_http.py"]
```

Crie `apps/mcp/.dockerignore` antes do primeiro build:

```dockerignore
.git
.env
.env.*
!.env.example
.venv
.venv-*
venv
__pycache__
*.pyc
node_modules
```

Assim COPY não inclui credenciais nem o ambiente Windows na imagem. Adicione somente as bibliotecas realmente usadas ao `apps/mcp/requirements.txt`.

No `compose.yaml`, acrescente o exemplo abaixo **dentro de services**, no mesmo nível de `n8n:`:

```yaml
  mcp:
    build:
      context: ./apps/mcp
      dockerfile: Dockerfile
    restart: unless-stopped
    env_file:
      - ./apps/mcp/.env
    environment:
      # Somente se o código do seu MCP lê esta variável.
      RAW_DOCUMENTS_DIR: /data/documentos/raw
    ports:
      - "127.0.0.1:8001:8001"
    volumes:
      - ./apps/mcp:/app
      - ./dados/documentos:/data/documentos
    networks:
      - aula
```

O contexto `./apps/mcp` faz COPY ler **apps/mcp/requirements.txt**. O env_file aponta para **apps/mcp/.env**. Não existe outra lista central nem necessidade do argumento PYTHON_REQUIREMENTS.

O exemplo pressupõe que server_http.py inicia Streamable HTTP em `0.0.0.0:8001`, caminho `/mcp`. Confira a configuração e o SDK do servidor real. A escuta em 0.0.0.0 permite conexões pela rede do contêiner.

O volume `./apps/mcp:/app` facilita editar código durante a aula. Após editar Python, reinicie o serviço se ele não possuir recarga automática. Dependências são instaladas na imagem: editar requirements.txt exige reconstrução.

## 6. Incorporar o backend de um RAG

Crie `apps/rag/Dockerfile` e um .dockerignore adaptado ao exemplo anterior:

```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN python -m pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "backend.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

O pacote uvicorn e todas as bibliotecas do backend devem constar no requirements.txt desse RAG. Pacotes com dependências nativas podem exigir instruções adicionais no Dockerfile.

Acrescente em services:

```yaml
  rag:
    build:
      context: ./apps/rag
      dockerfile: Dockerfile
    restart: unless-stopped
    env_file:
      - ./apps/rag/.env
    ports:
      - "127.0.0.1:8000:8000"
    volumes:
      - ./apps/rag/backend:/app/backend
      - ./dados/documentos:/app/documentos
    networks:
      - aula
```

O código desse exemplo deve apontar para `/app/documentos/raw` e `/app/documentos/gold`. Confira config.py e o comportamento real: uma variável declarada no Compose só produz efeito se o app a lê.

O n8n chama `http://rag:8000`. O Windows usa `http://localhost:8000`. A primeira inicialização pode baixar modelos de embeddings; aguarde a API ficar pronta. Configure a persistência desses modelos conforme o projeto RAG.

## 7. Incorporar um frontend Next.js

Crie `apps/rag/frontend/Dockerfile`:

```dockerfile
FROM node:22-bookworm-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev", "--", "--hostname", "0.0.0.0"]
```

Este exemplo é para desenvolvimento, pressupõe package-lock.json e um script dev compatível com Next.js. Adapte a versão Node.js e o gerenciador ao projeto.

Crie `apps/rag/frontend/.dockerignore`:

```dockerignore
.git
.env
.env.*
!.env.example
node_modules
.next
dist
npm-debug.log
```

Acrescente em services:

```yaml
  rag-frontend:
    build:
      context: ./apps/rag/frontend
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      # Funciona se o next.config do projeto lê BACKEND_URL.
      BACKEND_URL: http://rag:8000
    ports:
      - "127.0.0.1:3000:3000"
    volumes:
      - ./apps/rag/frontend:/app
      - rag_frontend_node_modules:/app/node_modules
      - rag_frontend_next:/app/.next
    depends_on:
      - rag
    networks:
      - aula
```

Na seção `volumes:` da raiz do Compose, junto de n8n_data, acrescente:

```yaml
  rag_frontend_node_modules:
  rag_frontend_next:
```

node_modules e .next ficam em volumes Linux separados dos arquivos Windows. Ao mudar o lockfile com um volume já existente, atualize também suas dependências:

```powershell
docker compose run --rm --no-deps rag-frontend npm ci
docker compose up -d --build rag-frontend
```

depends_on simples organiza o início, mas não garante que a API esteja pronta. Aguarde o backend ou implemente um healthcheck no projeto.

BACKEND_URL é exemplo de configuração de um proxy Next.js. Variáveis NEXT_PUBLIC_* são públicas no navegador; não use segredos nelas. O navegador Windows não resolve o nome interno rag; configure o proxy ou a URL publicada apropriada.

## 8. Ler e editar o YAML

| Campo | Significado no exemplo MCP |
|---|---|
| mcp | Nome do serviço e seu DNS na rede interna |
| build.context | Pasta enviada para construir a imagem |
| build.dockerfile | Dockerfile relativo ao contexto |
| env_file | Arquivo local com variáveis de execução |
| environment | Variáveis do Compose; prevalecem sobre env_file |
| ports | Porta Windows à esquerda; contêiner à direita |
| volumes | Pasta Windows relativa à esquerda; Linux à direita |
| networks | Rede que o serviço compartilha |
| command | Opcional: substitui o CMD do Dockerfile |
| restart | Política de reinício do serviço |

Use espaços e mantenha a indentação. Preserve services, volumes e networks da raiz. Acrescente cada serviço no nível correto e não duplique uma chave no mesmo nível.

**Compose não lê requirements.txt automaticamente:** o Dockerfile copia esse arquivo e chama pip. O .env é configuração de execução, não uma lista de pacotes.

## 9. Construir e atualizar

Na raiz do laboratório:

```powershell
# Confere o YAML sem exibir valores de credenciais.
docker compose config --quiet

# Constrói as imagens dos apps e inicia os serviços declarados.
docker compose up -d --build
docker compose ps
docker compose logs -f mcp
```

Ctrl+C encerra somente a visualização dos logs.

Para atualizar um app:

```powershell
git -C apps/mcp pull
docker compose up -d --build mcp
```

| O que mudou? | Ação |
|---|---|
| Pacotes Python ou Dockerfile | docker compose up -d --build mcp |
| Código copiado para a imagem | Reconstruir com --build |
| Código montado por volume | Recarga do app ou docker compose restart mcp |
| Conteúdo do .env | docker compose up -d --force-recreate mcp |
| Novo serviço no YAML | docker compose up -d --build NOME_DO_SERVICO |
| Lockfile Node em volume existente | npm ci no serviço, como no exemplo Next.js |

restart sozinho não recarrega variáveis de env_file. O build usa cache quando as dependências não mudaram.

## 10. Rede e dados

| Origem da chamada | Destino |
|---|---|
| Navegador/Python no Windows → n8n | http://localhost:5678 |
| Navegador no Windows → frontend | http://localhost:3000 |
| n8n → RAG | http://rag:8000 |
| n8n → MCP | http://mcp:8001/mcp |

Dentro de um contêiner, localhost aponta para ele mesmo. O nome do serviço alcança outro contêiner na rede aula.

| Pasta no Windows | n8n e MCP dos exemplos | RAG do exemplo |
|---|---|---|
| dados/documentos/raw | /data/documentos/raw | /app/documentos/raw |
| dados/documentos/silver | /data/documentos/silver | /app/documentos/silver |
| dados/documentos/gold | /data/documentos/gold | /app/documentos/gold |

Essas pastas não processam arquivos por si mesmas. No exercício RAG, o próprio backend move para gold depois de concluir a persistência. Acrescentar silver não realiza ETL automaticamente.

Local File Trigger está liberado no n8n inicial; Execute Command permanece excluído. Verifique eventos em pastas Windows compartilhadas. Um observador que chama Webhook é alternativa. Controle eventos repetidos antes da ingestão.

## 11. Persistência, backup e mudança de computador

### Onde o n8n guarda os dados

No `compose.yaml` inicial, o volume nomeado `n8n_data` é montado em `/home/node/.n8n` dentro do contêiner. Na configuração padrão com SQLite, o arquivo `database.sqlite` guarda workflows, credenciais criptografadas, dados de execução e outras informações da instância. Se o n8n estiver configurado para PostgreSQL, o banco fica fora desse arquivo e precisa de backup próprio. [Banco de dados do n8n](https://docs.n8n.io/deploy/host-n8n/configure-n8n/choose-n8ns-database/).

No computador do professor, o Docker Desktop com WSL apresentou este caminho no Windows:

~~~text
\\wsl.localhost\docker-desktop\mnt\docker-desktop-disk\data\docker\volumes\laboratorio-ia-docker_n8n_data\_data
~~~

Ali está o `database.sqlite` do caso observado. **Esse caminho é apenas um exemplo**: versão do Docker Desktop, configuração e nome do projeto Compose podem alterá-lo. `n8n_data` é o nome no YAML; `laboratorio-ia-docker_n8n_data` é o nome do volume que o Docker gerou para esse projeto.

Prefira acessar o conteúdo pelo próprio Docker para realizar o backup, em vez de modificar diretamente a área interna do Docker Desktop no WSL. Um `git clone` não copia o volume. O procedimento usa um contêiner auxiliar para ler o volume, seguindo a abordagem de [backup de volumes do Docker](https://docs.docker.com/engine/storage/volumes/#back-up-restore-or-migrate-data-volumes).

### Por que copiar o volume inteiro

O banco SQLite não é necessariamente o único arquivo necessário. O n8n gera uma chave de criptografia na pasta `.n8n`; essa chave é usada para proteger as credenciais guardadas no banco. Copiar apenas `database.sqlite` para outra instância pode recuperar workflows sem permitir o uso das credenciais. Se a instalação usa `N8N_ENCRYPTION_KEY` definida fora do volume, guarde essa mesma chave separadamente em local privado e configure-a no destino. [Chave de criptografia do n8n](https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/set-a-custom-encryption-key/).

O SQLite também pode manter arquivos auxiliares, como `database.sqlite-wal`, quando o modo WAL está ativo. Para uma cópia consistente neste laboratório, **aguarde as execuções terminarem, pare o n8n e copie o volume completo**, incluindo arquivos auxiliares que existam. [SQLite: arquivo WAL](https://www.sqlite.org/wal.html).

O backup do volume pode conter **workflows, dados de execução, credenciais criptografadas e a própria chave**. O arquivo `.tar.gz` apenas compacta: **ele não é criptografado**. Trate-o como dado sensível: use armazenamento privado com proteção adequada; não publique no Git, em repositório público ou no grupo da turma. A pasta `backups/` do template está no `.gitignore`.

### Criar uma cópia no computador de origem

O exemplo abaixo usa PowerShell na pasta que contém o `compose.yaml` e pressupõe o serviço `n8n` deste template, com SQLite e um único contêiner escritor. Aguarde os workflows em execução terminarem. Se houver outros contêineres ou processos que escrevem nesse mesmo volume, pare-os também antes da cópia.

**O comando de parada abaixo interrompe o serviço n8n**; execute-o quando puder interromper a automação. O Docker Desktop precisa estar iniciado. A primeira execução pode baixar a imagem auxiliar `alpine:3.20`.

~~~powershell
# Pare somente o n8n e aguarde o encerramento.
docker compose stop -t 60 n8n
if ($LASTEXITCODE -ne 0) { throw "Falha ao parar o n8n. Backup não iniciado." }

# Descubra o contêiner, confirme que está parado e identifique o volume real.
$n8nContainerId = docker compose ps --all --quiet n8n
if (-not $n8nContainerId) { throw "Contêiner n8n não encontrado neste Compose." }

$n8nEstado = docker inspect --format '{{.State.Status}}' $n8nContainerId
if ($LASTEXITCODE -ne 0 -or $n8nEstado -notin @("exited", "created")) {
    throw "O n8n não está parado. Verifique o estado antes de copiar."
}

$n8nMontagensJson = docker inspect --format '{{json .Mounts}}' $n8nContainerId
if ($LASTEXITCODE -ne 0) { throw "Não foi possível inspecionar as montagens do n8n." }
$n8nMontagens = $n8nMontagensJson | ConvertFrom-Json
$n8nVolume = ($n8nMontagens | Where-Object { $_.Destination -eq "/home/node/.n8n" -and $_.Type -eq "volume" }).Name
if ($LASTEXITCODE -ne 0 -or -not $n8nVolume) {
    throw "Volume n8n_data não encontrado; confira a montagem do Compose."
}

# Crie um arquivo com horário no nome, evitando substituir backups anteriores.
$backupDir = Join-Path (Get-Location).Path "backups"
New-Item -ItemType Directory -Path $backupDir -Force | Out-Null
$backupNome = "n8n-" + (Get-Date -Format "yyyyMMdd-HHmmss-fff") + ".tar.gz"
if (Test-Path -LiteralPath (Join-Path $backupDir $backupNome)) {
    throw "O arquivo de backup já existe. Escolha outro nome."
}

# O volume de origem é montado somente para leitura.
docker run --rm --mount "type=volume,src=$n8nVolume,dst=/origem,readonly" --mount "type=bind,src=$backupDir,dst=/backup" alpine:3.20 tar -czf "/backup/$backupNome" -C /origem .
if ($LASTEXITCODE -ne 0) { throw "Falha na criação do backup. O n8n continua parado." }

# Confirme que o arquivo pode ser lido e liste seus nomes, sem exibir conteúdo.
docker run --rm --mount "type=bind,src=$backupDir,dst=/backup,readonly" alpine:3.20 tar -tzf "/backup/$backupNome"
if ($LASTEXITCODE -ne 0) { throw "Falha ao ler o backup. O n8n continua parado." }

Write-Host "Backup criado em: $backupDir\$backupNome"
docker compose start n8n
if ($LASTEXITCODE -ne 0) { throw "Backup criado, mas não foi possível iniciar o n8n." }
~~~

A listagem deve incluir `./database.sqlite` e, na configuração padrão com chave gerada pelo n8n, `./config`. A listagem verifica que o arquivo pode ser lido; **ela não substitui um teste de restauração**. Se o comando falhar, confira a mensagem antes de reiniciar o n8n com `docker compose start n8n`.

Copie o `.tar.gz` para o pendrive ou armazenamento privado. Copie também, separadamente, os dados necessários de `dados/documentos`, os projetos em `apps`, seus `.env` e qualquer chave definida fora do volume. Esses itens **não estão incluídos** no backup de `n8n_data`.

### Restaurar em outro Docker Desktop

Este procedimento restaura o **estado completo** da instância, inclusive contas, workflows e credenciais criptografadas. Use apenas um arquivo criado por você e de origem confiável. Se o computador de destino já tiver workflows ou credenciais que precise preservar, faça backup deles primeiro: o procedimento abaixo se recusa a extrair em um volume que já contenha arquivos.

Se a intenção for levar somente os workflows, outra opção é exportá-los pela interface do n8n e configurar as credenciais novamente no destino. A exportação de workflows não é uma cópia completa do banco nem das credenciais.

1. Instale e inicie o Docker Desktop no computador de destino. Clone o template e entre na sua pasta no PowerShell.
2. Crie uma pasta local `backups` e copie o `.tar.gz` para ela por um meio privado.
3. No script abaixo, substitua **apenas** o nome fictício de `$backupNome` pelo nome real do seu arquivo. Use a mesma versão da imagem n8n usada ao criar o backup; avalie atualizações posteriormente.

Ao iniciar a instância restaurada, workflows que estavam ativos podem disparar novamente. Antes de executar o último comando, revise se há gatilhos ou integrações que podem gerar efeitos duplicados nas duas máquinas.

~~~powershell
$backupNome = "n8n-AAAAMMDD-HHMMSS-mmm.tar.gz"
$backupDir = Join-Path (Get-Location).Path "backups"
if (-not (Test-Path -LiteralPath (Join-Path $backupDir $backupNome) -PathType Leaf)) {
    throw "Arquivo de backup não encontrado na pasta backups."
}

# Pare o serviço se já existir e crie o contêiner e o volume sem iniciar o n8n.
docker compose stop -t 60 n8n
if ($LASTEXITCODE -ne 0) { throw "Falha ao parar o n8n do destino." }
docker compose create n8n
if ($LASTEXITCODE -ne 0) { throw "Falha ao criar o contêiner do destino." }

$n8nContainerId = docker compose ps --all --quiet n8n
if (-not $n8nContainerId) { throw "Contêiner n8n do destino não encontrado." }
$n8nEstado = docker inspect --format '{{.State.Status}}' $n8nContainerId
if ($LASTEXITCODE -ne 0 -or $n8nEstado -notin @("exited", "created")) {
    throw "O n8n do destino não está parado."
}
$n8nMontagensJson = docker inspect --format '{{json .Mounts}}' $n8nContainerId
if ($LASTEXITCODE -ne 0) { throw "Não foi possível inspecionar as montagens do n8n." }
$n8nMontagens = $n8nMontagensJson | ConvertFrom-Json
$n8nVolume = ($n8nMontagens | Where-Object { $_.Destination -eq "/home/node/.n8n" -and $_.Type -eq "volume" }).Name
if ($LASTEXITCODE -ne 0 -or -not $n8nVolume) {
    throw "Volume do destino não encontrado."
}

# Confira o volume antes de extrair. Nenhum outro processo deve escrever nele.
$conteudoVolume = docker run --rm --mount "type=volume,src=$n8nVolume,dst=/destino,readonly" alpine:3.20 ls -A /destino
if ($LASTEXITCODE -ne 0) { throw "Não foi possível conferir o volume do destino." }
if ($conteudoVolume) { throw "ABORTADO: o volume de destino não está vazio." }

# O volume vazio recebe o backup; nenhum arquivo existente é apagado.
docker run --rm --mount "type=volume,src=$n8nVolume,dst=/destino" --mount "type=bind,src=$backupDir,dst=/backup,readonly" alpine:3.20 tar -xzf "/backup/$backupNome" -C /destino
if ($LASTEXITCODE -ne 0) { throw "Restauração interrompida. O n8n continua parado; confira o volume e o arquivo." }

docker compose start n8n
if ($LASTEXITCODE -ne 0) { throw "Volume restaurado, mas não foi possível iniciar o n8n." }
~~~

Se o volume não estiver vazio, **não o apague para forçar esse procedimento**: preserve o estado existente e planeje a migração. Se uma chave externa `N8N_ENCRYPTION_KEY` era usada na origem, configure a mesma chave no destino sem publicá-la.

Abra `http://localhost:5678`, entre com a conta restaurada e confira workflows, uma credencial de teste e os caminhos utilizados. Os workflows que estavam ativos no backup podem voltar a executar quando o n8n iniciar; revise gatilhos, URLs e possíveis efeitos duplicados antes de colocar as duas instâncias em operação.

Os dados compartilhados em `dados/documentos` e os projetos em `apps` precisam ser restaurados separadamente. Reconecte integrações que dependam de permissões ou caminhos da máquina anterior. Ao atualizar o n8n, siga o procedimento de atualização da versão escolhida.

## 12. Git do laboratório e Git dos apps

Os diretórios dos apps clonados são ignorados pelo template por padrão. Faça commit e push de cada aplicação dentro de seu próprio repositório.

O Compose personalizado pertence ao seu laboratório. Para reunir todos os apps num repositório novo, escolha uma estratégia explícita, como submódulos ou subtree. Clones Git aninhados não viram automaticamente arquivos comuns do repositório principal.

## 13. Parar o ambiente

```powershell
docker compose stop
```

Para remover contêineres e a rede do Compose preservando volumes nomeados:

```powershell
docker compose down
```

Não acrescente -v se quiser preservar os dados. O mecanismo Docker Desktop precisa estar ativo; a janela pode ficar minimizada.

## Documentação oficial

- [Compose: contexto e construção](https://docs.docker.com/reference/compose-file/build/)
- [Dockerfile e .dockerignore](https://docs.docker.com/build/building/best-practices/)
- [Compose: env_file e serviços](https://docs.docker.com/reference/compose-file/services/)
- [Rede do Compose](https://docs.docker.com/compose/how-tos/networking/)
- [n8n com Docker Compose](https://docs.n8n.io/deploy/host-n8n/install-options/install-using-docker-compose/)
