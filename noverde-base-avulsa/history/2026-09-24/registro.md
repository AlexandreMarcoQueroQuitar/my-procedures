# Registro de execução - base avulsa Noverde

Data: 2026-09-24

## Contexto

Foi formatada uma base da Noverde para o contrato de base avulsa do
`campaign_engine` do repositório `qq`. A origem utilizada foi o CSV extraído
em `PA-QueroQuitar-16092026`, e o resultado foi gravado como
`noverde_140926.csv`.

Nenhum CPF ou e-mail é reproduzido neste registro.

## Parâmetros e decisões

- Credor/provider: `noverde`.
- Marcação: `noverde_202609`.
- Telefone: vazio.
- `bypass_sms`: `0`.
- `bypass_email`: `1`.
- E-mail: coluna `email` da origem.
- Dados adicionais: `{"cf_credit_amount": "R$ 1.234,56"}`.
- Valor: coluna `amount` da origem.
- Arredondamento: centavos com `ROUND_HALF_UP`.
- Células com dois e-mails separados por `;`: usado o primeiro valor.
- Linhas sem e-mail: preservadas; não criam contato novo.

## Confirmação no código

Foi confirmado que:

- `creditor` é usado como `DataProvider.Provider` para contato novo;
- registros de base avulsa recebem `no_debt=true`;
- o engine `bases_avulsas_email` aceita pessoa sem dívida;
- não existe bloqueio por ausência de dívida no pseudo-credor `noverde`;
- a oitava coluna trafega como dados adicionais da base avulsa.

## Resultado

- Linhas processadas: 1.097.242.
- Linhas sem e-mail na origem: 5.502.
- Células com múltiplos e-mails: 3.
- Valores `amount` vazios ou inválidos: 0.
- Valores com mais de duas casas decimais: 324.594.
- Linhas com quantidade incorreta de colunas: 0.
- JSONs inválidos: 0.
- Valores fora do prefixo monetário `R$ `: 0.
- Tamanho final: 110.621.997 bytes.
- SHA-256 final:
  `82f7852b7223a8e14fb0a4ce564c1ca809e6d128d52a72316bbfe1e83cdd8927`.

## Observação operacional

O formato não oferece uma flag para proibir o encaminhamento à fila de SMS.
Mesmo com telefone vazio e `bypass_sms=0`, uma pessoa já presente no cache de
telefone pode ser encaminhada à fila `bases_avulsas_sms`. O arquivo não cria
telefone novo.
