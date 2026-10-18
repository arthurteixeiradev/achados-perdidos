# Relatório técnico

## 1. Dockerfile

O Dockerfile monta a imagem de web com Python, as dependências e o código da aplicação.

| Instrução | Explicação |
| --- | --- |
| `FROM python:3.12-slim` | Usa a imagem oficial enxuta do Python 3.12. |
| `WORKDIR /app` | Define a pasta de trabalho dentro da imagem. |
| `COPY requirements.txt .` | Copia primeiro a lista de dependências. |
| `RUN pip install --no-cache-dir -r requirements.txt` | Instala as dependências sem guardar o cache de downloads do pip. |
| `COPY app.py .` | Copia o código depois da instalação das dependências. |
| `RUN useradd ... && mkdir ... && chown ...` | Cria o usuário `appuser`, a pasta `/app/uploads` e dá a ele a propriedade dessa pasta. |
| `USER appuser` | Define que a aplicação vai rodar sem root. |
| `EXPOSE 5000` | Informa a porta usada dentro do contêiner. |
| `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]` | Inicia o Gunicorn na porta 5000, em formato exec. `app:app` indica o objeto Flask no arquivo `app.py`. |

Essa ordem aproveita o cache de camadas. Se mudar apenas `app.py`, a etapa de instalação das dependências pode ser reutilizada. O cache do Docker continua funcionando mesmo com `--no-cache-dir`, que vale para o pip.

O `.dockerignore` exclui `.env`, `__pycache__/` e `*.pyc`. O contexto de build é a pasta `app/`.

Quando o volume de fotos é criado vazio, o Docker copia para ele o conteúdo e as permissões da pasta de montagem da imagem. Nos testes, os arquivos ficaram com dono `appuser`, e `whoami` retornou esse mesmo usuário.

## 2. Docker Compose

| Bloco | Explicação |
| --- | --- |
| `web.build` | Constrói a imagem usando `./app`. |
| `web.environment` | Passa host, porta, banco, usuário, senha e diretório de uploads para o Python. `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` viram `DB_NAME`, `DB_USER` e `DB_PASSWORD`. |
| `web.ports` | Publica a porta 5000 de web como 8080 no computador. |
| `web.volumes` | Monta `fotos` em `/app/uploads`. |
| `web.depends_on` | Espera db ficar saudável antes de iniciar web. |
| `db.image` | Usa `postgres:16-alpine`. |
| `db.environment` | Recebe as três variáveis `POSTGRES_*` do `.env` para inicializar o banco. |
| `db.volumes` | Guarda o banco em `dados_banco:/var/lib/postgresql/data` e monta `db/init.sql` em `/docker-entrypoint-initdb.d/init.sql`, somente para leitura. |
| `db.healthcheck` | Usa `pg_isready` a cada 5 segundos, com timeout de 5 segundos e até 10 tentativas. O teste por TCP em `127.0.0.1` espera o servidor final do PostgreSQL. |
| `networks` | Liga os dois serviços à bridge `rede_interna`. |
| `restart` | Usa `unless-stopped` nos dois serviços. O Docker pode reiniciá-los, respeitando uma parada explícita. |
| `volumes` | Declara `dados_banco` e `fotos`, que existem além da vida dos contêineres. |

No healthcheck, `$$` deixa a variável ser expandida pelo shell do contêiner. O Compose lê os valores do `.env`; a senha não aparece no código nem no Dockerfile. O banco não tem porta publicada no computador.

Web usa `db:5432` para acessar o PostgreSQL. A rede própria permite resolver o nome `db`, mesmo que o IP mude. O caminho das fotos é `/app/uploads`, configurado em `UPLOAD_DIR` e montado no volume `fotos`.

## 3. Caminho de um POST /itens com foto

1. O curl lê a imagem no computador e envia um formulário com `descricao`, `local` e `foto` para `localhost:8080/itens`.
2. Na configuração principal, a porta 8080 encaminha a conexão para o Gunicorn na porta 5000 de web.
3. Flask executa `cadastrar()`, lê os campos e verifica se descrição e local foram informados.
4. O código gera um nome com UUID e mantém a extensão do arquivo em minúsculas.
5. A foto é salva em `/app/uploads`, dentro do volume `fotos`.
6. Psycopg abre uma conexão com `db:5432` e executa o INSERT parametrizado na tabela `itens`. O banco guarda a descrição, o local e o nome da foto.
7. Ao sair normalmente do bloco de conexão, a transação é confirmada. Os arquivos do PostgreSQL ficam no volume `dados_banco`.
8. A aplicação responde com o ID, o nome da foto e HTTP 201.
9. Na consulta `GET /itens`, o nome vira um caminho `/fotos/<nome>`. Ao abrir esse caminho, a aplicação lê o arquivo do volume e envia a imagem.

A aplicação salva a foto antes do INSERT. Se a gravação no banco falhar depois, pode sobrar um arquivo sem cadastro. Esse comportamento é do código fornecido e foi mantido.

## 4. Ciclo de vida dos volumes

| Etapa dos testes guiados | Itens | Fotos |
| --- | ---: | ---: |
| Três exemplos e o primeiro cadastro com foto | 4 | 1 |
| Mais dois cadastros com foto | 6 | 3 |
| Depois de `down` e `up -d` | 6 | 3 |
| Depois de `down -v` e `up -d` | 3 | 0 |
| Cadastro usado no backup e volta à configuração principal | 4 | 1 |

Com `down`, os contêineres e a rede foram removidos, mas os volumes permaneceram. Depois de subir de novo, os itens e as fotos continuaram disponíveis.

A inspeção mostrou os dados em `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data` e `/var/lib/docker/volumes/achados-perdidos_fotos/_data`. No Docker Desktop usado no teste, esses caminhos ficam no ambiente Linux do Docker.

Com `down -v`, os dois volumes foram apagados. A subida seguinte criou volumes vazios: o PostgreSQL executou o `init.sql` novamente, inseriu os três exemplos e o diretório de fotos ficou vazio.

Depois foi cadastrada uma foto para o teste de backup. O arquivo `fotos-reteste.tar.gz` foi restaurado em um volume novo. A comparação não encontrou diferenças, e os hashes SHA256 foram iguais. O volume temporário foi removido, mantendo o original.

As capturas estão em [RELATORIO_TESTES.md](RELATORIO_TESTES.md). A saída completa da inspeção de volumes e rede está no [arquivo 05](evidencias/05-volumes-rede-imagem.txt). As demais saídas complementares estão no [índice de evidências](evidencias/README.md).

Referências usadas: [imagem oficial PostgreSQL](https://hub.docker.com/_/postgres) e [ordem de inicialização no Compose](https://docs.docker.com/compose/how-tos/startup-order/).
