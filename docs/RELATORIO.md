# Relatório técnico

## (a) Instruções e ordem do Dockerfile

| Instrução de `app/Dockerfile` | Finalidade e posição |
| --- | --- |
| `FROM python:3.12-slim` | Usa a variante oficial enxuta do Python 3.12, conforme a Tarefa 2. |
| `WORKDIR /app` | Define onde os arquivos serão copiados e onde Gunicorn importará `app.py`. |
| `COPY requirements.txt .` | Copia primeiro a lista fornecida, isolando a camada de dependências das mudanças de código. |
| `RUN pip install --no-cache-dir -r requirements.txt` | Instala Flask 3.0.3, psycopg binary 3.2.3 e Gunicorn 23.0.0, sem guardar cache de downloads do pip. |
| `COPY app.py .` | Só depois das dependências copia o código imutável do apêndice. Uma mudança apenas nesse arquivo permite reutilizar a instalação anterior. |
| `RUN useradd --create-home appuser ...` | Cria o usuário sem privilégios, cria `/app/uploads` e transfere a propriedade desse diretório para `appuser:appuser`. |
| `USER appuser` | Faz o Gunicorn e a aplicação executarem sem root. |
| `EXPOSE 5000` | Documenta a porta interna; a publicação no host depende do Compose. |
| `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]` | Inicia Gunicorn em formato exec, em todas as interfaces do contêiner na porta 5000; importa o objeto `app` do módulo `app.py`. |

O volume `fotos`, quando novo e vazio, recebe o conteúdo e as permissões do diretório de montagem da imagem pelo comportamento padrão dos volumes Docker. Por isso `appuser` consegue gravar nele sem executar a aplicação como root. A [evidência 02](evidencias/02-funcionais.txt) registra `whoami=appuser` e o upload; a [05](evidencias/05-volumes-rede-imagem.txt) mostra os arquivos pertencendo a esse usuário. O `.dockerignore` exclui `.env`, `__pycache__/` e `*.pyc` do contexto `./app`.

## (b) Blocos do docker-compose.yml

| Bloco | Configuração e efeito |
| --- | --- |
| `services.web.build` | Constrói a imagem usando `./app` como contexto e o Dockerfile dessa pasta. |
| `services.web.environment` | Injeta `DB_HOST`, `DB_PORT` e `UPLOAD_DIR` do `.env`; traduz `POSTGRES_DB`, `POSTGRES_USER` e `POSTGRES_PASSWORD` para `DB_NAME`, `DB_USER` e `DB_PASSWORD`, nomes esperados pelo código. |
| `services.web.ports` | Publica somente `8080:5000`, permitindo o acesso HTTP pelo host. |
| `services.web.volumes` | Monta `fotos` em `/app/uploads`, exatamente o `UPLOAD_DIR` configurado. |
| `services.web.depends_on` | A condição `service_healthy` exige que o healthcheck de `db` tenha sucesso antes do início de web. |
| `services.db.image` | Usa `postgres:16-alpine`, sem imagem própria do banco. |
| `services.db.environment` | Recebe as três variáveis `POSTGRES_*` do `.env`, utilizadas pela imagem oficial ao inicializar um banco vazio. |
| `services.db.volumes` | Monta `dados_banco` em `/var/lib/postgresql/data`; monta `./db/init.sql` em `/docker-entrypoint-initdb.d/init.sql:ro`. O SQL permanece um arquivo do host, somente leitura no contêiner. |
| `services.db.healthcheck` | Executa `pg_isready -h 127.0.0.1 -U ... -d ...` a cada 5 segundos, com timeout de 5 segundos e até 10 tentativas. `$$` preserva a expansão das variáveis para o shell do contêiner. O teste TCP evita confundir o servidor temporário por socket da inicialização com o servidor final. |
| `networks` dos serviços | Liga ambos a `rede_interna`, permitindo a resolução DNS de `db` por web. |
| `restart` dos serviços | `unless-stopped` reinicia os contêineres após falhas ou reinício do daemon, respeitando a parada explícita. |
| `networks.rede_interna` | Declara uma rede bridge própria. Não há publicação da porta 5432 do banco. |
| `volumes` | Declara `dados_banco` e `fotos` como volumes nomeados independentes da vida dos contêineres. |

O `.env` é lido pelo Compose para interpolação; não é copiado para a imagem nem versionado. `.env.example` contém apenas valores fictícios e as mesmas seis chaves. Não há senha literal no Dockerfile, no Compose ou no código. O código usa `/app/uploads` e `db:5432` como padrões; `DB_NAME`, `DB_USER` e `DB_PASSWORD` precisam ser fornecidos. O Gunicorn define explicitamente a porta 5000 no Dockerfile, pois o código fornecido não chama `app.run()`.

A [evidência 01](evidencias/01-subida.txt) mostra o build, `db` saudável e o início subsequente de web; a [02](evidencias/02-funcionais.txt) confirma conexão e serviços; a [05](evidencias/05-volumes-rede-imagem.txt) registra rede, montagens físicas dos volumes e imagem. Nesse Docker, `image ls` informa 219 MB de uso em disco e 53,8 MB de conteúdo; `image inspect` informa `Size=53758665` bytes. São medidas distintas exibidas pelo armazenamento de imagens desse Engine.

## (c) Fluxo completo de POST /itens com foto

1. `curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@docs/evidencias/oculos.jpg" http://localhost:8080/itens` lê a imagem do host e envia uma requisição multipart HTTP à porta 8080.
2. O mapeamento `8080:5000` entrega a conexão ao Gunicorn em `web`, participante da `rede_interna`.
3. Gunicorn encaminha a requisição à rota `cadastrar()` do Flask. O código lê `descricao` e `local` em `request.form` e o arquivo em `request.files`.
4. A validação já fornecida retorna 400 se descrição ou local faltarem. Havendo arquivo com nome, o código extrai a extensão em minúsculas e gera um nome usando `uuid.uuid4().hex`.
5. `foto.save()` grava `/app/uploads/<uuid>.jpg`. Esse caminho está no volume `fotos`, e a escrita é feita com o usuário `appuser`. O arquivo fica no armazenamento do Docker, fora da camada descartável do contêiner.
6. `conectar()` usa psycopg e a configuração `DB_*`; o DNS do Docker resolve `db` na rede bridge. A conexão SQL chega ao PostgreSQL na porta interna 5432.
7. O INSERT parametrizado registra descrição, local e nome da foto na tabela `itens`, retornando o ID. A saída normal do contexto da conexão confirma a transação. O PostgreSQL mantém seus arquivos em `/var/lib/postgresql/data`, no volume `dados_banco`.
8. Flask responde JSON com `id` e nome de `foto`, HTTP 201. O POST não retorna o caminho completo da rota.
9. `GET /itens` consulta a tabela e converte o nome em `/fotos/<nome>`. O navegador faz esse GET; `send_from_directory()` lê a imagem do volume e a devolve por Gunicorn e pela porta publicada.

A [evidência 02](evidencias/02-funcionais.txt) contém o cadastro real e hashes SHA256 idênticos da imagem original e recebida; a [03](evidencias/03-foto-http.txt) mostra HTTP 200 e `Content-Type: image/jpeg`. A aplicação grava o arquivo antes da transação SQL, exatamente como no apêndice; não foi acrescentado mecanismo de transação entre banco e sistema de arquivos.

## (d) Ciclo de vida dos volumes comprovado na Tarefa 5

| Etapa executada | Itens | Fotos | Evidência |
| --- | ---: | ---: | --- |
| Volume inicial com os três exemplos e cadastro da Tarefa 4 | 4 | 1 | [02-funcionais.txt](evidencias/02-funcionais.txt) |
| Mais dois objetos cadastrados com foto | 6 | 3 | [04-persistencia-recriacao.txt](evidencias/04-persistencia-recriacao.txt) |
| `down` seguido de `up -d` | 6 | 3 | [04-persistencia-recriacao.txt](evidencias/04-persistencia-recriacao.txt) |
| Inspeção dos volumes e arquivos | 6 | 3 | [05-volumes-rede-imagem.txt](evidencias/05-volumes-rede-imagem.txt) |
| Antes de `down -v` | 6 | 3 | [08-persistencia-remocao.txt](evidencias/08-persistencia-remocao.txt) |
| `down -v` seguido de `up -d` | 3 | 0 | [08-persistencia-remocao.txt](evidencias/08-persistencia-remocao.txt) |

`down` removeu contêineres e rede, preservando ambos os volumes. A listagem manteve os IDs e as fotos; a foto baixada novamente teve o mesmo SHA256. A imagem não precisou ser reconstruída e o SQL inicial não foi reexecutado sobre os dados existentes.

A inspeção identificou `/var/lib/docker/volumes/achados-perdidos_dados_banco/_data` e `/var/lib/docker/volumes/achados-perdidos_fotos/_data`. Esses caminhos pertencem ao host Linux do daemon; neste Windows com Docker Desktop, ficam dentro da VM Linux e não diretamente em `C:\`.

`down -v` removeu os dois volumes: o `volume inspect` seguinte retornou `no such volume`. Ao subir novamente, os volumes vazios foram criados, as fotos anteriores desapareceram e o PostgreSQL executou novamente o bind mount `init.sql`; os logs registram `CREATE TABLE` e `INSERT 0 3`. Os três exemplos foram inseridos de novo; os objetos cadastrados durante os testes foram perdidos, como esperado.

Comportamentos de inicialização e prontidão conferidos na [imagem oficial PostgreSQL](https://hub.docker.com/_/postgres) e na [documentação de ordem de inicialização do Compose](https://docs.docker.com/compose/how-tos/startup-order/). O ambiente principal não contém migrações nem persistência adicional.
