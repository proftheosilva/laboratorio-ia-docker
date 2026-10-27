# Evolução do laboratório por incorporação de projetos

O estado inicial executa somente n8n. Os serviços crescem quando o aluno incorpora aplicações reais desenvolvidas e testadas.

## Ciclo de incorporação

1. Desenvolver e testar a aplicação em seu repositório.
2. Clonar em apps/NOME_DO_PROJETO.
3. Restaurar o .env privado.
4. Criar ou adaptar Dockerfile e .dockerignore do próprio app.
5. Adicionar a entrada em services com contexto, variáveis, portas, volumes e rede.
6. Construir e conferir a aplicação no Docker.
7. Versionar código no repositório do app e configurações de integração no laboratório personalizado.

## Aplicações possíveis

- MCP: SDK e pacotes declarados pelo servidor, com transporte compatível com o cliente.
- RAG: extração, chunking, embeddings e persistência definidos pelo projeto.
- Next.js: dependências do package.json e lockfile do frontend.
- ETL, agentes e memória: incorporados quando os projetos existirem e precisarem dessas capacidades.

Cada aplicação controla suas versões e dependências. Bancos ou serviços de memória só são adicionados se a arquitetura exigir. O laboratório pode usar Supabase e outros serviços remotos.

## Dados e comunicação

Os serviços compartilham a rede aula e se comunicam pelo nome do serviço. Monte dados/documentos somente nos serviços que precisam acessá-los.

raw, silver e gold apoiam a evolução do curso. Criar ou mover pastas não constitui ETL. As transformações e as regras de promoção pertencem ao aplicativo.

## Antes da aula

- Conferir caminhos, portas e dependências de cada app.
- Revisar .env e .dockerignore sem publicar segredos.
- Conferir inicialização, processamento e persistência.
- Definir o tratamento de eventos repetidos e falhas.
- Planejar dados portáteis e restauração de workflows n8n.

Consulte o [README principal](../README.md) para comandos e exemplos.
