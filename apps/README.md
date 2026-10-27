# Aplicações incorporadas pelos alunos

Esta pasta recebe os projetos reais desenvolvidos e testados em aula. O template inicia somente o n8n.

1. Clone seu repositório em `apps/NOME_DO_PROJETO`.
2. Restaure o `.env` privado na raiz desse app.
3. Crie ou adapte o Dockerfile e o .dockerignore dentro do projeto.
4. Mantenha dependências no requirements.txt do projeto Python ou no package.json e lockfile do projeto Node.js.
5. Acrescente seu serviço no compose.yaml da raiz e conecte-o à rede aula.
6. Confira com `docker compose config --quiet` e inicie com `docker compose up -d --build`.

Os exemplos completos de MCP, backend RAG e Next.js estão no [README principal](../README.md).

Não há Dockerfile genérico nem lista central de pacotes. Um projeto com frontend e backend pode usar dois serviços e dois Dockerfiles.

Cada app mantém seu próprio repositório. Os diretórios clonados são ignorados pelo Git do template por padrão. As regras do laboratório não substituem o .gitignore de cada aplicação.
