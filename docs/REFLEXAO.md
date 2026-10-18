# Reflexão

## Q1 — Como web encontra db?

Web e db estão na mesma rede bridge, definida em `docker-compose.yml`.  
O DNS do Docker resolve o nome `db` para o IP do contêiner do banco.  
Assim, a aplicação continua usando `db:5432` mesmo após uma mudança de IP.  
Na bridge padrão de `docker run`, essa resolução por nome não acontece automaticamente; seria preciso usar IP, configurar a resolução ou recorrer ao legado `--link`.  
A rede está nas capturas e na evidência 05; a diferença entre as bridges está na [documentação Docker](https://docs.docker.com/engine/network/drivers/bridge/).

## Q2 — Volume nomeado ou bind mount?

O volume nomeado é criado e administrado pelo Docker; o bind mount usa um caminho do computador.  
Usamos bind mount para `db/init.sql` porque ele já é um arquivo do projeto e só precisa ser lido pelo banco.  
Por isso sua montagem tem `:ro`, que impede alterações por dentro do contêiner.  
Os arquivos do PostgreSQL e as fotos mudam durante o uso e precisam sobreviver à recriação, então ficaram nos volumes `dados_banco` e `fotos`.  
Os caminhos físicos e os arquivos aparecem na evidência 05 e nas capturas da inspeção.

## Q3 — Alterar init.sql depois da primeira subida funciona?

Não basta editar `init.sql` e executar `up -d` de novo.  
A imagem PostgreSQL só executa os scripts de inicialização quando o diretório de dados está vazio.  
No teste com `down -v`, o volume foi apagado e o script voltou a criar os três exemplos, como mostram as capturas da remoção e da nova subida.  
Para mudar uma tabela sem perder os dados, é preciso fazer backup e aplicar o SQL necessário, como um `ALTER TABLE`, no banco existente.  
Isso pode ser feito com `docker compose exec db psql -U <usuario> -d <banco>`; o comportamento da inicialização está na [imagem oficial](https://hub.docker.com/_/postgres).

## Q4 — Por que esperar service_healthy?

Um processo iniciado ainda pode estar preparando o banco e recusar conexões.  
O `depends_on` simples só organiza a ordem de início dos contêineres.  
No nosso Compose, `pg_isready` verifica a disponibilidade, e `service_healthy` faz web esperar esse resultado.  
A saída da subida mostra db saudável antes do início de web; isso também está na evidência 01.  
Essa espera vale para a inicialização e não reinicia web automaticamente se o banco falhar depois ([documentação Compose](https://docs.docker.com/compose/how-tos/startup-order/)).

## Q5 — Por que copiar requirements.txt antes de app.py?

As dependências mudam com menos frequência que o código da aplicação.  
No Dockerfile, copiamos `requirements.txt`, instalamos os pacotes e depois copiamos `app.py`.  
Se só `app.py` mudar, o Docker pode reaproveitar a camada que instalou as dependências.  
Se a lista de pacotes mudar, essa instalação e as etapas seguintes precisam ser refeitas.  
O `--no-cache-dir` evita o cache de downloads do pip, mas não desliga o cache de camadas do Docker; a ordem aparece na evidência 01.

## Q6 — Por que não publicar 5432? Como proteger as fotos?

A aplicação acessa o banco por `db:5432` na rede Docker, então não precisa publicar essa porta no computador.  
O Compose publica apenas 8080 para web, como aparece nas saídas de `docker compose ps`.  
Isso reduz a exposição direta do banco, mas as fotos de documentos ainda podem conter dados pessoais.  
Considerando a proteção desses dados e o art. 46 da [LGPD](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm), proponho autenticação e autorização para consultar itens e fotos.  
A segunda medida seria HTTPS para proteger o envio das imagens; essas medidas são propostas da reflexão e não foram acrescentadas à aplicação.

## Q7 — O que aconteceu com três réplicas?

O comando `docker compose up -d --scale web=3` falhou porque a porta 8080 já estava ocupada pela primeira instância.  
As capturas mostram o erro; depois voltamos para uma instância com `--scale web=1`.  
Para usar três réplicas, seria preciso retirar a publicação fixa de cada web e colocar um proxy/balanceador distribuindo as requisições entre elas.  
No mesmo host, elas podem usar o volume `fotos`: cada upload ganha um UUID, o que torna a colisão de nomes muito improvável, mantendo permissões compatíveis.  
Isso não garante uma transação única entre foto e banco; uma falha depois de salvar o arquivo pode deixá-lo sem registro.  
Em hosts diferentes, o volume local não é compartilhado automaticamente; seria necessário armazenamento compartilhado ou de objetos.

## 5.1 — Equivalência em nuvem

| Peça local | Microsoft Azure | AWS | Google Cloud |
| --- | --- | --- | --- |
| Contêiner web | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Contêiner db + volume dados_banco | Azure Database for PostgreSQL Flexible Server, com armazenamento gerenciado | Amazon RDS for PostgreSQL, com armazenamento gerenciado | Cloud SQL for PostgreSQL, com armazenamento gerenciado |
| Volume fotos | Azure Blob Storage | Amazon S3 | Cloud Storage |
| rede_interna | Azure Virtual Network, sub-redes e regras de segurança | Amazon VPC, sub-redes e security groups | VPC, sub-redes e regras de firewall |
| .env com segredos | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem local | Azure Container Registry | Amazon Elastic Container Registry | Artifact Registry |

A maior mudança seria o armazenamento das fotos. Hoje `app.py` grava, lista e lê arquivos em `/app/uploads`. Blob Storage, S3 e Cloud Storage trabalham com objetos por API, então essa escolha exigiria adaptar essas operações e o acesso às imagens. O banco gerenciado continuaria usando PostgreSQL, com ajustes de conexão e operação. Se a intenção fosse manter o código sem mudanças, seria preciso avaliar uma opção de armazenamento de arquivos montável e compatível com o serviço de execução escolhido.

Fontes da tabela: [comparação entre provedores](https://docs.cloud.google.com/docs/get-started/aws-azure-gcp-service-comparison), [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview), [Amazon ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html), [Cloud Run](https://docs.cloud.google.com/run/docs/overview/what-is-cloud-run) e [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction).
