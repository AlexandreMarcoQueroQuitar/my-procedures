# Registro de execução - base avulsa Noverde

Data: 2026-09-28

## Contexto

Correção do procedimento após identificação de que o campo de valor correto
é `gc_credit_amount`. Foi acrescentado `gc_credit_inst`, derivado de
`max_period`. O registro de 2026-09-24 foi preservado como histórico da
execução anterior.

## Entrada e saída

- Origem: `/home/queroquitar/Downloads/PA-QueroQuitar-16092026/bq-results-20260916-125635-1789563404662.csv`.
- Saída: `noverde-base-avulsa/.workspace/noverde_160926.csv`.
- Credor/provider: `noverde`.
- Marcação: `noverde_202609`.
- Campos adicionais: `gc_credit_amount` recebe `amount` formatado em reais;
  `gc_credit_inst` recebe `max_period` como texto.
- Arredondamento do valor: centavos com `ROUND_HALF_UP`.
- E-mail: primeiro valor da coluna `email` quando separados por `;`.

## Validação e resultado

- Linhas de origem e saída: 1.097.242.
- Linhas sem e-mail: 5.502; células com múltiplos e-mails: 3.
- `max_period` vazio ou inválido: 0.
- Largura incorreta, campos fixos incorretos, JSON inválido, valor monetário
  inválido e período inválido na saída: 0 em cada categoria.
- Comparação linha a linha com a origem de documento, e-mail, valor formatado
  e número de parcelas: 1.097.242 correspondências.
- Tamanho da saída: 136.692.817 bytes.
- SHA-256: `f26aeb03f497467b9d0876d0c9603a0c3c620453e1a1938eaae9856a702db769`.

A saída permanece em `.workspace/` para entrega local e não deve ser commitada.
Nenhum CPF ou e-mail foi incluído neste registro.
