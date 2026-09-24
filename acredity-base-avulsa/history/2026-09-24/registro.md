# Registro de execução - base avulsa Acredity

Data: 2026-09-24

## Contexto

Foi convertido o arquivo `AN_QUERO_QUITAR 1.txt` para uma base avulsa do
`campaign_engine`, selecionando somente registros com
`vl_margem_disponivel_clt > 0`.

Nenhum CPF, nome ou outro dado pessoal é reproduzido neste registro.

## Parâmetros e decisões

- Credor: `acredity`.
- Marcação: `aprovados_acredity_0826`.
- Saída: `aprovados_acredity_0826_base_avulsa.csv`.
- Formato: `CPF;acredity;aprovados_acredity_0826`.
- Cabeçalho na saída: não.
- Telefone/e-mail: não fornecidos pela origem e não incluídos.
- Flags de bypass: não incluídas.
- Coluna `data`: não incluída.
- Margem: usada apenas como filtro numérico e não copiada para a saída.

## Resultado

- Linhas de dados na origem: 1.175.987.
- Margem maior que zero: 288.280.
- Margem vazia: 887.707.
- Margem igual a zero: 0.
- Margem negativa: 0.
- Margem inválida: 0.
- CPFs inválidos entre os aprovados: 0.
- CPFs duplicados entre os aprovados: 0.
- Linhas finais: 288.280.
- Tamanho final: 12.972.600 bytes.
- SHA-256:
  `5cf12ec6344eb2d3a2c0faa0a2fbb18ae982f75f7765943482925ebf88fbd0e5`.

## Comportamento operacional

Como a base possui apenas três colunas, nenhum contato novo é criado. O QQ
encaminha a pessoa apenas aos canais em que ela já consta nos caches de
contato. A aceitação sem dívida continua habilitada para as filas de base
avulsa.

## Teste do procedimento

O procedimento documentado foi executado integralmente contra o arquivo de
origem indicado, usando uma saída temporária independente.

Resultado do teste:

- linhas lidas: 1.175.987;
- linhas aprovadas: 288.280;
- linhas com quantidade incorreta de colunas: 0;
- campos fixos divergentes: 0;
- documentos fora do formato: 0;
- documentos duplicados: 0;
- SHA-256 da saída de teste:
  `5cf12ec6344eb2d3a2c0faa0a2fbb18ae982f75f7765943482925ebf88fbd0e5`;
- comparação byte a byte com o arquivo entregue: idêntica.
