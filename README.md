# Achados & Perdidos

Este projeto coloca a API Flask e o PostgreSQL em contêineres Docker. Os itens ficam no banco e as fotos ficam em um volume separado. Os arquivos `app.py`, `requirements.txt` e `init.sql` são os fornecidos na atividade.

## O que precisa estar instalado

Docker com suporte a contêineres Linux e Docker Compose. No Windows, o Docker Desktop deve estar aberto. A porta 8080 precisa estar disponível.

Os comandos abaixo são para o Prompt de Comando do Windows (cmd), dentro da pasta `achados-perdidos`.

## Configuração

Na primeira execução, copie o modelo:

```cmd
copy .env.example .env
```

No PowerShell, use `Copy-Item .env.example .env`. Se o `.env` já estiver configurado, mantenha o arquivo existente.

Edite o `.env` e troque a senha fictícia por uma senha local. As chaves são:

| Chave | Para que serve |
| --- | --- |
| `POSTGRES_DB` | Nome do banco; também vira `DB_NAME` em web. |
| `POSTGRES_USER` | Usuário do banco; também vira `DB_USER` em web. |
| `POSTGRES_PASSWORD` | Senha local; também vira `DB_PASSWORD` em web. |
| `DB_HOST` | Use `db`, o nome do serviço PostgreSQL. |
| `DB_PORT` | Use `5432`, a porta interna do banco. |
| `UPLOAD_DIR` | Use `/app/uploads`, onde o volume de fotos é montado. |

O `.env` não deve ser enviado nem versionado. O `.env.example` pode ser entregue, pois só tem valores fictícios. Mudar a senha no `.env` não altera a senha de um banco que já foi inicializado.

## Subir o ambiente

```cmd
docker compose up -d --build
docker compose ps
```

Espere db ficar `healthy` e web aparecer em execução. A aplicação fica em `http://localhost:8080`. Na primeira inicialização de um volume vazio, o banco recebe três itens de exemplo.

## Testar

```cmd
curl.exe http://localhost:8080/health
curl.exe http://localhost:8080/itens
curl.exe "http://localhost:8080/itens?local=biblio"
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@docs/evidencias/oculos.jpg" http://localhost:8080/itens
curl.exe http://localhost:8080/estatisticas
docker compose logs web --tail 20
docker compose exec web whoami
```

O health deve retornar `ok` e `conectado`. O usuário deve ser `appuser`. A imagem `oculos.jpg` é um arquivo de teste incluído no projeto.

Para abrir a foto, consulte `GET /itens`, copie o caminho `/fotos/<nome>` e acrescente `http://localhost:8080` antes dele. O POST retorna o nome do arquivo; a listagem retorna o caminho da foto.

| Método | Rota | O que faz |
| --- | --- | --- |
| GET | `/health` | Testa a conexão com o banco; retorna 503 se ela falhar. |
| GET | `/itens` | Lista os itens, dos mais recentes para os mais antigos. |
| GET | `/itens?local=texto` | Filtra por parte do local, sem diferenciar maiúsculas e minúsculas. |
| POST | `/itens` | Recebe `descricao`, `local` e uma `foto` opcional. Retorna 201 no cadastro e 400 se faltar um campo obrigatório. |
| GET | `/fotos/<nome>` | Devolve uma foto salva no volume. |
| GET | `/estatisticas` | Conta os itens no banco e os arquivos no diretório de fotos. |

## Parar e remover

Para parar e iniciar os mesmos contêineres:

```cmd
docker compose stop
docker compose start
```

Para remover os contêineres e a rede, mantendo os dados:

```cmd
docker compose down
```

Para apagar também o banco e as fotos:

```cmd
docker compose down -v
```

Depois de `down -v`, a próxima subida cria volumes vazios e executa novamente o `init.sql`.

## Documentação

- [Arquitetura](docs/ARQUITETURA.md).
- [Relatório técnico](docs/RELATORIO.md).
- [Q1 a Q7 e tabela de equivalência em nuvem](docs/REFLEXAO.md).
- [Relatório dos testes com capturas de tela](docs/RELATORIO_TESTES.md).
- [Índice das evidências](docs/evidencias/README.md).
- [Como executar os três bônus](docs/bonus/README.md).

Nos testes de 17/10/2026, seis itens e três fotos sobreviveram à recriação dos contêineres. Depois de remover os volumes, voltaram os três exemplos e nenhuma foto. Os bônus nginx, Adminer e backup/restauração também foram testados.

Ao terminar os testes guiados, a configuração principal ficou em execução com quatro itens e uma foto. As evidências incluem as capturas dos testes e as saídas complementares dos comandos.

A entrega deve ser feita pelo repositório ou por um ZIP sem `.env`.
