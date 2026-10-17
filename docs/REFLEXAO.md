# Reflexão — questões Q1 a Q7

## Q1 — Resolução do nome db

`docker-compose.yml` conecta web e db à bridge própria `rede_interna`.  
O DNS embutido do Docker resolve o nome do serviço `db` para seu IP nessa rede.  
Assim, `DB_HOST=db` continua válido mesmo se a recriação mudar o endereço IP.  
A [evidência 05](evidencias/05-volumes-rede-imagem.txt) identifica os participantes da rede; a conexão funciona na evidência 02.  
Na bridge padrão de `docker run`, não há essa resolução automática entre nomes: seria preciso IP, configuração explícita ou o legado `--link`, conforme a [documentação Docker](https://docs.docker.com/engine/network/drivers/bridge/).

## Q2 — Volume nomeado e bind mount

Um volume nomeado é armazenamento administrado pelo Docker, independente do diretório do projeto.  
Um bind mount projeta um arquivo ou diretório específico do host dentro do contêiner.  
O Compose usa `./db/init.sql:...:ro` porque o SQL é um arquivo fornecido e versionado, apenas lido na inicialização.  
`dados_banco` guarda os arquivos mutáveis do PostgreSQL sem depender de um caminho escolhido no Windows.  
`fotos` segue a mesma lógica; os caminhos físicos e a persistência aparecem nas evidências [04](evidencias/04-persistencia-recriacao.txt) e [05](evidencias/05-volumes-rede-imagem.txt).

## Q3 — Alteração posterior do init.sql

Editar `db/init.sql` e repetir `up -d` não aplica a mudança a um banco já inicializado.  
A imagem oficial só executa `/docker-entrypoint-initdb.d/` ao criar o banco em um diretório de dados vazio.  
A [evidência 08](evidencias/08-persistencia-remocao.txt) mostra a execução do script após a remoção dos volumes e o retorno aos três exemplos.  
Para preservar dados, faça backup do banco e aplique a alteração explicitamente com SQL, por exemplo `ALTER TABLE`, via `docker compose exec db psql -U <usuario> -d <banco>`.  
`down -v` recriaria a estrutura, mas apagaria os dados; esse não é o procedimento para evoluir o banco existente ([imagem oficial](https://hub.docker.com/_/postgres)).

## Q4 — Prontidão do banco

`depends_on` simples estabelece ordem de início, mas um processo iniciado pode ainda estar inicializando o PostgreSQL.  
Nesse intervalo, a primeira conexão de web poderia falhar e `/health` retornaria 503.  
Nosso Compose testa `pg_isready` por TCP e usa `condition: service_healthy`.  
A [evidência 01](evidencias/01-subida.txt) registra db saudável antes do início de web.  
A condição trata a prontidão inicial; não faz supervisão contínua da dependência após iniciar web ([documentação Compose](https://docs.docker.com/compose/how-tos/startup-order/)).

## Q5 — Cache das camadas

Em `app/Dockerfile`, copiamos `requirements.txt`, instalamos as dependências e só então copiamos `app.py`.  
Se somente `app.py` mudar, o arquivo de requisitos e as etapas anteriores continuam iguais e podem reutilizar o cache do build.  
O Docker reconstrói a partir do `COPY app.py`, sem reinstalar as dependências dessa camada anterior.  
Se `requirements.txt` mudar, a instalação e as etapas seguintes precisam ser refeitas.  
`--no-cache-dir` evita o cache de downloads do pip dentro da imagem; não desativa o cache de camadas Docker, cuja ordem aparece na [evidência 01](evidencias/01-subida.txt).

## Q6 — Porta do banco e proteção de dados

Web já alcança `db:5432` pela rede própria, portanto publicar essa porta não atende a uma necessidade do ambiente.  
Nosso Compose publica apenas `8080:5000`; a [evidência 02](evidencias/02-funcionais.txt) mostra o banco sem mapeamento de porta para o host.  
Isso reduz a exposição direta do banco; fotos de documentos continuam exigindo controle de acesso na API.  
Relacionando à proteção de dados pessoais e ao art. 46 da [LGPD](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm), proponho autenticação com autorização para consultar itens e fotos.  
Como segunda medida, proponho HTTPS para proteger o transporte das fotos; ambas são propostas de reflexão, sem alterações ao código ou ao escopo obrigatório.

## Q7 — Três réplicas de web

Executei `docker compose up -d --scale web=3`: a [evidência 06](evidencias/06-escala.txt) registra `port is already allocated`.  
Cada réplica tenta reservar a mesma porta 8080 do host; a primeira continua rodando e as adicionais não conseguem iniciar.  
Seria necessário retirar a publicação fixa das réplicas e encaminhar o tráfego por um proxy/balanceador configurado para distribuir entre as três instâncias.  
No mesmo host Docker, elas podem compartilhar `fotos`: os UUIDs tornam improvável a colisão e cada upload grava seu próprio arquivo; permissões precisam permanecer compatíveis.  
Isso não torna foto e INSERT uma transação atômica; falha entre ambos pode deixar arquivo órfão, como permite o código fornecido.  
Em hosts diferentes, o volume local não é automaticamente compartilhado; seria necessário armazenamento compartilhado ou de objetos. Restaurei `--scale web=1` após o teste.

# 5.1 — Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
| --- | --- | --- | --- |
| Contêiner web | Azure Container Apps | Amazon ECS com AWS Fargate | Cloud Run |
| Contêiner db + volume dados_banco | Azure Database for PostgreSQL Flexible Server, com armazenamento gerenciado | Amazon RDS for PostgreSQL, com armazenamento gerenciado | Cloud SQL for PostgreSQL, com armazenamento gerenciado |
| Volume fotos | Azure Blob Storage | Amazon S3 | Cloud Storage |
| rede_interna | Azure Virtual Network (VNet), sub-redes e regras de segurança | Amazon VPC, sub-redes e security groups | VPC, sub-redes e regras de firewall |
| Arquivo .env (segredos) | Azure Key Vault | AWS Secrets Manager | Secret Manager |
| Imagem construída localmente | Azure Container Registry | Amazon Elastic Container Registry (ECR) | Artifact Registry |

As equivalências indicam responsabilidades na arquitetura, não uma migração automática. Os serviços de execução foram conferidos nas documentações de [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview), [Amazon ECS/Fargate](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html) e [Cloud Run](https://docs.cloud.google.com/run/docs/overview/what-is-cloud-run). Banco, armazenamento, rede, segredos e registros têm correspondentes na [comparação oficial entre provedores](https://docs.cloud.google.com/docs/get-started/aws-azure-gcp-service-comparison); veja também [PostgreSQL no Azure](https://learn.microsoft.com/en-us/azure/postgresql/overview).

A peça que mais muda é o volume `fotos`. Hoje `app.py` usa `foto.save()`, `os.listdir()` e `send_from_directory()` sobre `/app/uploads`. Blob Storage, S3 e Cloud Storage oferecem objetos por API, não um volume POSIX equivalente ao volume local: essa opção de migração exigiria adaptar gravação, contagem e leitura das fotos, além de permissões e URLs. O banco gerenciado continuaria falando PostgreSQL, com mudanças principalmente de conexão e operação. Para preservar o código sem alterações numa futura migração, seria necessário avaliar armazenamento de arquivos montável e compatível com a plataforma escolhida; a tabela apresenta a alternativa de objetos, sem implementá-la nesta atividade ([Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction)).
