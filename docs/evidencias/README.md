# Evidências reais

Execução em 17/10/2026, fuso America/Sao_Paulo, Docker Desktop com engine Linux 29.7.2 e Compose v5.4.0. Os logs dos contêineres e cabeçalhos HTTP usam UTC. As saídas são reais, guardadas em texto; não são exemplos de resultados esperados. Nos arquivos 02 em diante, `>` identifica o comando e `Exit code` registra seu código de saída. A evidência 01 contém a saída de `docker compose up -d --build`.

| Arquivo | Legenda |
| --- | --- |
| [01-subida.txt](01-subida.txt) | Build e primeira subida do projeto, com banco saudável antes de web. |
| [02-funcionais.txt](02-funcionais.txt) | Versões, serviços, health, listagem, filtro, cadastro com foto, estatísticas, logs, usuário e hashes da foto. |
| [03-foto-http.txt](03-foto-http.txt) | Cabeçalhos reais de GET da foto: HTTP 200 e conteúdo JPEG. |
| [04-persistencia-recriacao.txt](04-persistencia-recriacao.txt) | Seis itens e três fotos mantidos após down/up, com IDs e hash preservados. |
| [05-volumes-rede-imagem.txt](05-volumes-rede-imagem.txt) | Inspeções de volumes e rede, arquivos, portas e tamanho da imagem web. |
| [06-escala.txt](06-escala.txt) | Tentativa de três réplicas falha pela reserva da porta 8080; escala volta a uma instância. |
| [07-bonus-backup.txt](07-bonus-backup.txt) | Backup tar.gz e restauração em volume novo com conteúdo e hashes iguais. |
| [08-persistencia-remocao.txt](08-persistencia-remocao.txt) | down -v remove volumes; a nova subida reinicializa três exemplos e zero fotos. |
| [09-bonus-proxy-adminer.txt](09-bonus-proxy-adminer.txt) | Nginx assume 8080, Adminer inicia só com debug e a configuração principal é restaurada. |
| [10-foto-navegador.txt](10-foto-navegador.txt) | Comando e registro do Chrome headless abrindo a URL real da foto na validação em pasta nova. |
| [10-foto-navegador.png](10-foto-navegador.png) | Captura real do navegador exibindo o JPEG devolvido pela API. |
| [11-do-zero.txt](11-do-zero.txt) | Pasta nova sem .env original sobe após copiar o modelo, com volumes próprios e banco conectado. |
| [12-validacao-estatica.txt](12-validacao-estatica.txt) | Comparação dos três arquivos fornecidos com o PDF, checagem de estrutura, referências e ausência da senha local na entrega. |
| [oculos.jpg](oculos.jpg) | Imagem sintética de óculos usada como arquivo de entrada nos testes de upload. |
| [foto-retornada.jpg](foto-retornada.jpg) | Arquivo efetivamente recebido pela rota de fotos na Tarefa 4. |
| [foto-apos-recriar.jpg](foto-apos-recriar.jpg) | Arquivo recebido pela mesma rota após recriar os contêineres na Tarefa 5. |
| [fotos.tar.gz](fotos.tar.gz) | Backup real das três fotos do volume antes do teste destrutivo. |

As estatísticas observadas foram `4/1` após a Tarefa 4, `6/3` após mais dois cadastros, `6/3` após recriação e `3/0` após remoção dos volumes e nova subida, na ordem itens/fotos. A captura do navegador foi feita em uma segunda pasta, `validacao-do-zero`, com projeto e volumes separados; seu POST criou uma foto nova e não alterou os volumes principais. Os UUIDs dessa captura, portanto, diferem dos da primeira execução.

Falhas intencionais: a evidência 06 contém o conflito de porta solicitado por Q7; a 08 contém `no such volume` após a exclusão. Na 05, `docker compose port db 5432` termina sem mapeamento válido (`invalid IP:0` nesta versão); o isolamento é confirmado principalmente pela ausência de publicação em `docker compose ps` e pelo Compose entregue. Pulls, IDs e horários refletem o ambiente da execução e podem mudar em outra máquina.
