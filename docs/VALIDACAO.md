# Validação contra G1-CN-2N.pdf

Fonte autoritativa: as nove páginas do PDF fornecido. Validação realizada em 17/10/2026. Os arquivos `app/app.py`, `app/requirements.txt` e `db/init.sql` foram comparados ao texto do Apêndice A, removendo somente a margem de impressão e a quebra de página. Não foram alterados imports, funções, validações, rotas, versões ou SQL.

O Git foi inicializado após a preparação da solução; os commits foram organizados por etapa do projeto, mantendo as datas reais de criação.

## Requisitos funcionais e não funcionais

| Requisito | Resultado verificado | Arquivo / evidência |
| --- | --- | --- |
| RF01 | `/health`: `status=ok`, `banco=conectado`. | [02](evidencias/02-funcionais.txt), [11](evidencias/11-do-zero.txt) |
| RF02 | Três itens iniciais; filtro `local=biblio` retorna o item da Biblioteca. | [02](evidencias/02-funcionais.txt) |
| RF03 | POST multipart cadastra descrição, local e foto; `foto` permanece opcional no código fornecido. | [app.py](../app/app.py), [02](evidencias/02-funcionais.txt) |
| RF04 | Foto servida com HTTP 200, JPEG e hash igual; estatísticas refletem itens e arquivos. | [02](evidencias/02-funcionais.txt), [03](evidencias/03-foto-http.txt), [captura](evidencias/10-foto-navegador.png) |
| RNF01 | `docker compose up -d --build` funciona após copiar o modelo de ambiente, também em pasta nova. | [01](evidencias/01-subida.txt), [11](evidencias/11-do-zero.txt) |
| RNF02 | Seis itens e três fotos sobrevivem a down/up; download mantém hash. | [04](evidencias/04-persistencia-recriacao.txt) |
| RNF03 | Credenciais interpoladas de `.env`, ignorado por `.gitignore`; modelo fictício e ZIP sem `.env`. | [Compose](../docker-compose.yml), [.gitignore](../.gitignore), [.env.example](../.env.example), [12](evidencias/12-validacao-estatica.txt) |
| RNF04 | `USER appuser` e `whoami=appuser`; uploads funcionam com dono correto. | [Dockerfile](../app/Dockerfile), [02](evidencias/02-funcionais.txt), [05](evidencias/05-volumes-rede-imagem.txt) |
| RNF05 | Healthcheck `pg_isready` e `condition: service_healthy`; web inicia após db saudável. | [Compose](../docker-compose.yml), [01](evidencias/01-subida.txt) |

## Tarefas 1 a 6

| Tarefa | Conferência |
| --- | --- |
| 1 — Estrutura | Raiz `achados-perdidos/`, `app/`, `db/`, `docs/` e todos os arquivos exigidos; `.env` somente local e excluído da entrega. |
| 2 — Dockerfile | Python 3.12 slim oficial; requirements antes do código; pip sem cache; uploads criado e com dono sem root; USER; EXPOSE 5000; Gunicorn em `0.0.0.0:5000`, CMD exec; `.dockerignore` exclui os três padrões exigidos. |
| 3 — Compose | web e db; postgres:16-alpine; `POSTGRES_*` do `.env`; volume dados_banco e fotos nos caminhos corretos; init.sql bind ro; pg_isready; variáveis DB_* e UPLOAD_DIR; depends_on saudável; bridge rede_interna; unless-stopped em ambos; somente web publica 8080:5000. |
| 4 — Funcionais | Subida, ps, health, listagem, filtro, upload, estatísticas, foto e logs executados; imagem recebida e captura do navegador preservadas. |
| 5 — Persistência | Dois cadastros adicionais com foto; estatísticas antes das três etapas; down/up; volume ls/inspect; ls dos uploads; down -v e nova subida com retorno dos exemplos e perda das fotos. |
| 6 — Documentação | A1–A5 completos, com referências às evidências reais. |

## Artefatos A1 a A5

| Artefato | Entrega |
| --- | --- |
| A1 | [README](../README.md): objetivo, pré-requisitos, .env, subir, testar, parar, destruir e tabela das rotas. |
| A2 | [Diagrama Mermaid](ARQUITETURA.md): host, portas, contêineres, rede, volumes, caminhos e bind mount; elementos fora/dentro da rede identificados. |
| A3 | [RELATORIO.md](RELATORIO.md): instruções e ordem do Dockerfile, blocos Compose, POST com foto e ciclo dos volumes. |
| A4 | [docs/evidencias](evidencias/README.md): saídas das Tarefas 4 e 5, network inspect, image ls com tamanho e legenda por arquivo. |
| A5 | [REFLEXAO.md](REFLEXAO.md): Q1–Q7 em 3–8 linhas explícitas por questão, tabela 5.1 com seis peças e três provedores, e parágrafo sobre a maior mudança na migração. |

Q1 aborda DNS e bridge padrão; Q2, bind versus volume; Q3, inicialização e alteração sem perda; Q4, prontidão; Q5, cache; Q6, exposição do banco, LGPD e duas medidas propostas; Q7, tentativa real de escala, conflito de porta e compartilhamento de fotos. As propostas de Q6 e a migração discutida em 5.1 não foram acrescentadas à aplicação.

## Checklist da seção 7

- [x] `docker compose up -d --build` funciona em pasta nova após copiar `.env.example` para `.env` (evidência 11).
- [x] `/health` retorna `ok` e `conectado` (02 e 11).
- [x] Nenhuma senha real nos arquivos entregues; `.env` excluído do ZIP e ignorado no projeto (12).
- [x] Banco sem porta publicada no host (Compose e 02).
- [x] `docker compose exec web whoami` não retorna root (02 e 11).
- [x] Itens e fotos sobrevivem a down/up (04).
- [x] README com passo a passo executado do zero (11).
- [x] Diagrama com rede, volumes e portas (A2).
- [x] Relatório com partes (a), (b), (c) e (d) (A3).
- [x] Evidências nomeadas e legendadas (A4).
- [x] Sete questões e tabela de equivalência preenchidas (A5).

O ZIP está pronto para envio. A publicação de um repositório e o envio do link/ZIP no ambiente virtual continuam sendo atos de entrega do aluno; não foram executados nesta sessão.

## Situações que zeram critérios

| Situação do PDF | Verificação da solução |
| --- | --- |
| Senha real versionada | `.env` local não integra o ZIP nem o histórico Git; `.gitignore` o exclui; o modelo só contém valores fictícios. Os commits são locais, sem publicação. A busca pela senha local nos demais arquivos foi negativa (12). |
| Subida depende de passos manuais não documentados | Somente copiar/configurar `.env` e executar Compose; o teste em outra pasta usou o modelo fornecido, sem Python ou PostgreSQL no host (11). As ferramentas auxiliares de leitura do PDF e coleta das evidências não são dependências do ambiente entregue. |
| Evidências não correspondem ao código | O código e as configurações testados são os entregues. Os três bônus têm arquivos próprios e suas execuções estão identificadas (07 e 09). A captura em projeto separado está identificada (10 e 11). |

## Bônus da seção 6

| Desafio | Resultado |
| --- | --- |
| Proxy reverso (+0,4) | Override retira a publicação de web e nginx assume 8080; diagrama atualizado e HTTP 200 comprovado (09). |
| Backup (+0,3) | Contêiner temporário gera tar.gz; restauração em volume novo com diff e SHA256 iguais (07). |
| Adminer (+0,3) | `profiles: ["debug"]`, ausente sem profile e disponível em 8081 após ativação (09). |

O acesso HTTP ao painel foi verificado; o login interativo no Adminer não foi realizado. A imagem no navegador foi verificada em modo headless. A solução não inclui endpoints, validações ou dependências adicionais na aplicação.
