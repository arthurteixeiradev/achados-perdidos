# Arquitetura principal

```mermaid
flowchart TB
    subgraph HOST["Host: computador do usuário"]
        CLIENTE["Navegador / curl"]
        PORTA["Porta publicada no host: 8080"]
        SQL["Arquivo do host: db/init.sql"]
        subgraph ENGINE["Docker Engine Linux — VM do Docker Desktop no Windows"]
            subgraph REDE["rede_interna — bridge definida pelo usuário"]
                WEB["web — Python 3.12 / Flask / Gunicorn<br/>appuser — 0.0.0.0:5000"]
                DB["db — postgres:16-alpine<br/>5432 interno; sem publicação no host"]
                WEB -->|"SQL: db:5432 — DNS do Docker"| DB
            end
            FOTOS[("Volume fotos<br/>montagem no web: /app/uploads")]
            DADOS[("Volume dados_banco<br/>montagem no db: /var/lib/postgresql/data")]
        end
        CLIENTE -->|"HTTP: localhost:8080"| PORTA
        PORTA -->|"8080:5000"| WEB
        WEB -->|"grava e lê arquivos"| FOTOS
        DB -->|"grava e lê o banco"| DADOS
        SQL -->|"bind mount somente leitura<br/>/docker-entrypoint-initdb.d/init.sql"| DB
    end
```

Somente `web` e `db` são participantes da `rede_interna`. O cliente, a porta do host, o arquivo SQL e os volumes ficam fora dessa rede: arquivos e volumes são armazenamento montado, não participantes de rede. O host hospeda o Engine; no Docker Desktop deste teste, os volumes ficam no disco da VM Linux.

O nome `rede_interna` identifica a rede bridge do projeto; não configura a opção Docker `internal: true`. O isolamento exigido é obtido com rede própria e ausência de publicação da porta do banco. `5432/tcp` em `docker compose ps` indica a porta do contêiner, sem um mapeamento `host->contêiner`.

O Compose prefixou os nomes como `achados-perdidos_rede_interna`, `achados-perdidos_fotos` e `achados-perdidos_dados_banco`. A [inspeção real](evidencias/05-volumes-rede-imagem.txt) mostra os dois contêineres na rede. A arquitetura opcional com proxy está no [diagrama dos bônus](bonus/README.md).
