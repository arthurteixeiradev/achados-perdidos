# Achados & Perdidos

Ambiente da avaliação G1 de Computação em Nuvem: API Flask/Gunicorn, PostgreSQL 16, rede bridge própria e volumes persistentes para itens e fotos. Os três arquivos do Apêndice A foram transcritos sem mudanças no código.

## Pré-requisitos e configuração

Docker com suporte a contêineres Linux e Docker Compose v2 ou superior, daemon iniciado e porta 8080 disponível. Não é necessário instalar Python ou PostgreSQL no host. Os comandos abaixo usam PowerShell; no Linux/macOS, use `cp` no lugar de `Copy-Item` e `curl` no lugar de `curl.exe`.

Na raiz desta pasta (`achados-perdidos/`):

```powershell
Copy-Item .env.example .env
```

Edite `.env` e substitua a senha fictícia por uma senha local. Não versione nem entregue `.env`. Mantenha as mesmas seis chaves do modelo:

| Chave | Uso |
| --- | --- |
| `POSTGRES_DB` | Nome do banco; também alimenta `DB_NAME` do web. |
| `POSTGRES_USER` | Usuário do PostgreSQL; também alimenta `DB_USER` do web. |
| `POSTGRES_PASSWORD` | Senha local; também alimenta `DB_PASSWORD` do web. |
| `DB_HOST` | `db`, nome do serviço na rede Docker. |
| `DB_PORT` | `5432`, porta interna do banco. |
| `UPLOAD_DIR` | `/app/uploads`, caminho onde o volume `fotos` é montado. |

As credenciais de inicialização são aplicadas quando `dados_banco` está vazio. Alterar somente `.env` não troca a senha de um banco já inicializado.

## Subir e testar — Tarefa 4

```powershell
docker compose up -d --build
docker compose ps
curl.exe http://localhost:8080/health
curl.exe http://localhost:8080/itens
curl.exe "http://localhost:8080/itens?local=biblio"
curl.exe -F "descricao=Oculos de grau" -F "local=Bloco B" -F "foto=@docs/evidencias/oculos.jpg" http://localhost:8080/itens
curl.exe http://localhost:8080/estatisticas
docker compose logs web --tail 20
docker compose exec web whoami
```

Espere os dois serviços em execução e `db` com estado `healthy`; a API pode levar um instante após o início do Gunicorn. `/health` deve mostrar `status=ok` e `banco=conectado`; `whoami` deve retornar `appuser`. Em volumes novos, existem três itens sem fotos. Após o cadastro acima, são quatro itens e uma foto. `oculos.jpg` é uma imagem sintética de teste incluída nas evidências; pode ser substituída por uma imagem `.jpg` ou `.png` sua.

Em `GET /itens`, copie o caminho `/fotos/<nome>` do item cadastrado e abra `http://localhost:8080/fotos/<nome>` no navegador. O POST retorna somente o nome do arquivo no campo `foto`; a listagem retorna o caminho da rota.

| Método | Rota | Comportamento |
| --- | --- | --- |
| GET | `/health` | Verifica a conexão; retorna estado, banco e hostname; falha de conexão retorna 503. |
| GET | `/itens` | Lista os itens por data de criação decrescente. |
| GET | `/itens?local=texto` | Filtra o local por correspondência parcial, sem distinguir maiúsculas. |
| POST | `/itens` | Multipart com `descricao`, `local` e `foto` opcional; sucesso 201, campos obrigatórios ausentes 400. |
| GET | `/fotos/<nome>` | Devolve o arquivo de `/app/uploads`. |
| GET | `/estatisticas` | Retorna `itens_no_banco` e `fotos_no_volume`. |

## Persistência — Tarefa 5

Cadastre mais dois objetos com foto e registre as estatísticas antes e depois da recriação:

```powershell
curl.exe http://localhost:8080/estatisticas
curl.exe -F "descricao=Garrafa com foto" -F "local=Bloco B" -F "foto=@docs/evidencias/oculos.jpg" http://localhost:8080/itens
curl.exe -F "descricao=Casaco com foto" -F "local=Bloco B" -F "foto=@docs/evidencias/oculos.jpg" http://localhost:8080/itens
curl.exe http://localhost:8080/estatisticas
docker compose down
docker compose up -d
curl.exe http://localhost:8080/estatisticas
curl.exe http://localhost:8080/itens
```

No roteiro completo desde volumes vazios, os seis itens e as três fotos permanecem. Reabra também a URL de uma foto anterior.

Antes da inspeção, registre novamente as estatísticas. O prefixo dos recursos é derivado da pasta do projeto; os nomes abaixo correspondem à pasta `achados-perdidos`:

```powershell
curl.exe http://localhost:8080/estatisticas
docker volume ls
docker volume inspect achados-perdidos_dados_banco achados-perdidos_fotos
docker compose exec web ls -l /app/uploads
docker network inspect achados-perdidos_rede_interna
docker image ls
```

O passo seguinte apaga os dados e as fotos deste projeto:

```powershell
curl.exe http://localhost:8080/estatisticas
docker compose down -v
docker compose up -d
curl.exe http://localhost:8080/estatisticas
curl.exe http://localhost:8080/itens
```

Após a prontidão da API, voltam os três itens de exemplo e nenhuma foto: a imagem oficial executa `db/init.sql` na inicialização do novo volume vazio. Resultados reais: [evidências](docs/evidencias/README.md) e [relatório](docs/RELATORIO.md).

## Parar e destruir

```powershell
docker compose stop
docker compose start
```

Para remover os contêineres e a rede, preservando os dois volumes:

```powershell
docker compose down
```

Para remover também os volumes e apagar definitivamente os itens e fotos:

```powershell
docker compose down -v
```

## Documentação e entrega

- [Arquitetura](docs/ARQUITETURA.md).
- [Relatório técnico, partes (a) a (d)](docs/RELATORIO.md).
- [Evidências reais e legendas](docs/evidencias/README.md).
- [Q1–Q7 e tabela 5.1](docs/REFLEXAO.md).
- [Bônus separados: nginx, backup/restauração e Adminer](docs/bonus/README.md).
- [Validação dos requisitos e checklist da seção 7](docs/VALIDACAO.md).

Os testes foram executados em 17/10/2026 com Docker Desktop, engine Linux e Compose v5.4.0. O ambiente foi encerrado com `down`, preservando os volumes reinicializados com três itens de exemplo e zero fotos. As evidências registram os estados anteriores dos testes. Para entregar, use o repositório sem `.env` ou o ZIP fornecido, que exclui esse arquivo.
