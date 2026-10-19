# Evidências

Os testes foram realizados em 17/10/2026. As capturas e as saídas abaixo comprovam os testes funcionais, a persistência, a rede, o tamanho da imagem e os bônus.

| Arquivo | Legenda |
| --- | --- |
| [Relatório dos testes](../RELATORIO_TESTES.md) | Resumo de cada etapa com as capturas originais clicáveis. |
| [01-subida.txt](01-subida.txt) | Build e subida do ambiente, com db saudável antes de web. |
| [02-funcionais.txt](02-funcionais.txt) | Testes por curl, cadastro com foto, estatísticas, logs e usuário sem root. |
| [05-volumes-rede-imagem.txt](05-volumes-rede-imagem.txt) | Inspeção completa dos volumes e da rede, arquivos, portas e tamanho da imagem. |
| [11-do-zero.txt](11-do-zero.txt) | Subida em uma pasta nova, usando o modelo de ambiente e volumes próprios. |
| [oculos.jpg](oculos.jpg) | Imagem de teste usada nos uploads do README. |

Os textos são saídas originais de uma execução anterior do mesmo código. Por isso, IDs de contêineres e nomes UUID das fotos podem diferir dos prints. Os horários do terminal estão no fuso America/Sao_Paulo; logs e cabeçalhos HTTP podem usar UTC.

As capturas de persistência mostram seis itens e três fotos depois de down/up e três itens sem fotos depois de down -v. Ao terminar os bônus, o ambiente principal ficou com quatro itens e uma foto.

O erro de porta no teste de escala e o erro de volume inexistente após a remoção eram esperados. O timeout de worker observado nos logs também foi registrado; depois dele, /health respondeu normalmente.
