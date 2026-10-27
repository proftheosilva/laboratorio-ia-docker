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

## 11. Persistência e mudança de computador

O volume n8n_data guarda workflows, credenciais e estado **na máquina local**. Não acompanha git clone. Exporte workflows revisados e configure credenciais no novo ambiente, ou restaure um backup completo com sua chave de criptografia.

Em outro computador:

1. Instale e inicie Docker Desktop.
2. Clone o template ou seu laboratório personalizado.
3. Clone os apps e restaure seus .env e Dockerfiles.
4. Restaure suas entradas no Compose e os dados do pendrive/armazenamento privado.
5. Execute docker compose up -d --build.
6. Importe workflows e configure credenciais, ou restaure o estado completo do n8n.

Imagens, dependências e modelos precisam ser baixados na primeira máquina nova. Dados de aula não ficam no Git.

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
