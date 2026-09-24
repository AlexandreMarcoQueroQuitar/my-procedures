# Base avulsa Noverde

## Objetivo

Converter um CSV de leads da Noverde para o formato de base avulsa lido por
`qq/pkg/campaign_engine/action/read_standalone_lead.go`, usando o e-mail do
arquivo de origem e levando o valor de `amount` no JSON da coluna de dados.

O procedimento gera linhas sem cabeçalho no formato:

```text
documento;noverde;marcacao;;email;0;1;{"cf_credit_amount": "R$ 1.234,56"}
```

As oito posições são:

1. documento;
2. credor/provider (`noverde`);
3. marcação do lote, por exemplo `noverde_202609`;
4. telefone vazio;
5. e-mail da origem;
6. `bypass_sms=0`;
7. `bypass_email=1`;
8. JSON de dados adicionais.

## Comportamento confirmado no QQ

Antes de cada execução, confira se o contrato continua igual no branch que
será usado. Os arquivos relevantes são:

- `pkg/campaign_engine/action/read_standalone_lead.go`;
- `pkg/campaign_engine/flow/loading_standalone_leads_file.go`;
- `pkg/campaign_engine/flow/processing_fast_track_items.go`;
- `pkg/campaign_engine/flow/processing_person_to_activation.go`;
- `pkg/campaign_engine/config/rules.go`.

No comportamento verificado em setembro de 2026:

- a coluna 2 (`creditor`) vira `DataProvider.Provider` ao criar um e-mail;
- `read_standalone_lead` define `no_debt=true`;
- `bases_avulsas_email` aceita pessoas sem dívida;
- portanto, não é necessário haver dívida no pseudo-credor `noverde`;
- `bypass_email=1` permite encaminhar o registro mesmo quando pessoa/e-mail
  ainda não aparece no cache da base avulsa;
- a oitava coluna é propagada como `standaloneAdditionalData` e deve conter um
  objeto JSON sem `;`.

Importante: `bypass_sms=0` não significa bloqueio absoluto de SMS. Se a pessoa
já estiver no cache de telefone, o mesmo arquivo pode ser encaminhado também
para a fila `bases_avulsas_sms`. O formato atual não possui uma flag explícita
para proibir esse encaminhamento. Nenhum telefone novo é criado porque a
quarta coluna fica vazia.

## Pré-requisitos e entrada

O CSV de origem deve:

- estar em UTF-8, com ou sem BOM;
- usar vírgula como delimitador e respeitar aspas CSV;
- conter, no mínimo, as colunas `document`, `email` e `amount`;
- preservar documentos como texto, sem notação científica;
- ter `amount` numérico usando ponto como separador decimal.

Não exponha CPF ou e-mail em logs. Registre apenas contagens e validações
agregadas.

## Preparação segura

Crie as áreas locais, que não devem ser commitadas:

```bash
mkdir -p noverde-base-avulsa/.workspace noverde-base-avulsa/.secrets
```

Defina caminhos explícitos. Não sobrescreva a origem durante a conversão:

```bash
SOURCE_CSV=/caminho/absoluto/origem.csv
OUTPUT_CSV=/caminho/absoluto/noverde_base_avulsa.csv
MARK=noverde_202609
```

Quando o destino já existir, faça uma cópia recuperável antes da troca:

```bash
cp -- "$OUTPUT_CSV" noverde-base-avulsa/.workspace/arquivo-anterior.csv
```

## Conversão

Execute o script abaixo informando origem, saída temporária e marcação. Ele:

- faz leitura streaming, adequada para arquivos grandes;
- arredonda `amount` para centavos com `ROUND_HALF_UP`;
- formata o valor como moeda brasileira;
- usa o primeiro e-mail quando a célula contiver dois valores separados por
  `;`, porque o contrato aceita somente um e-mail na quinta coluna;
- preserva linhas sem e-mail, que não criarão contato novo;
- não escreve cabeçalho.

```bash
python3 - "$SOURCE_CSV" "$OUTPUT_CSV.tmp" "$MARK" <<'PY'
import csv
import json
import sys
from decimal import Decimal, InvalidOperation, ROUND_HALF_UP
from pathlib import Path

source = Path(sys.argv[1])
output = Path(sys.argv[2])
mark = sys.argv[3]

if not mark.startswith("noverde_"):
    raise SystemExit("a marcação deve começar com noverde_")

def format_brl(raw):
    try:
        value = Decimal(raw.strip()).quantize(
            Decimal("0.01"), rounding=ROUND_HALF_UP
        )
    except (AttributeError, InvalidOperation):
        raise ValueError("amount vazio ou inválido")
    formatted = f"{value:,.2f}"
    return "R$ " + formatted.replace(",", "X").replace(".", ",").replace("X", ".")

rows = 0
empty_email = 0
multiple_email = 0

with source.open("r", encoding="utf-8-sig", newline="") as src, output.open(
    "w", encoding="utf-8", newline=""
) as dst:
    reader = csv.DictReader(src)
    required = {"document", "email", "amount"}
    if not reader.fieldnames or not required.issubset(reader.fieldnames):
        raise SystemExit(
            "cabeçalho inválido; esperadas as colunas document, email e amount"
        )

    for row in reader:
        rows += 1
        document = (row.get("document") or "").strip()
        email = (row.get("email") or "").strip()

        if not document:
            raise SystemExit(f"document vazio na linha de dados {rows}")
        if any(c in document for c in ";\r\n"):
            raise SystemExit(f"document contém delimitador na linha de dados {rows}")

        email = email.replace("\r", "").replace("\n", "")
        if ";" in email:
            multiple_email += 1
            email = email.split(";", 1)[0].strip()
        if not email:
            empty_email += 1

        additional_data = json.dumps(
            {"cf_credit_amount": format_brl(row.get("amount"))},
            ensure_ascii=False,
        )
        dst.write(
            f"{document};noverde;{mark};;{email};0;1;{additional_data}\n"
        )

print(
    f"rows={rows} empty_email={empty_email} "
    f"multiple_email_first_used={multiple_email}"
)
PY
```

Se qualquer `amount` estiver ausente ou inválido, a execução deve falhar. Não
preencha valor financeiro por inferência.

## Validação antes da substituição

Valide todas as linhas e o JSON sem imprimir dados pessoais:

```bash
python3 - "$OUTPUT_CSV.tmp" <<'PY'
import json
import sys
from pathlib import Path

path = Path(sys.argv[1])
rows = bad_width = bad_fixed = bad_json = bad_currency = 0

with path.open("r", encoding="utf-8", newline="") as src:
    for line in src:
        rows += 1
        columns = line.rstrip("\r\n").split(";")
        if len(columns) != 8:
            bad_width += 1
            continue

        document, creditor, mark, phone, email, bypass_sms, bypass_email, data = columns
        if (
            not document
            or creditor != "noverde"
            or not mark.startswith("noverde_")
            or phone != ""
            or bypass_sms != "0"
            or bypass_email != "1"
        ):
            bad_fixed += 1

        try:
            obj = json.loads(data)
        except json.JSONDecodeError:
            bad_json += 1
            continue

        value = obj.get("cf_credit_amount")
        if not isinstance(value, str) or not value.startswith("R$ "):
            bad_currency += 1

print(
    f"rows={rows} bad_width={bad_width} bad_fixed={bad_fixed} "
    f"bad_json={bad_json} bad_currency={bad_currency}"
)

if any((bad_width, bad_fixed, bad_json, bad_currency)):
    raise SystemExit(1)
PY
```

Compare também a quantidade de linhas da origem e da saída:

```bash
python3 - "$SOURCE_CSV" "$OUTPUT_CSV.tmp" <<'PY'
import csv
import sys

with open(sys.argv[1], encoding="utf-8-sig", newline="") as src:
    source_rows = sum(1 for _ in csv.DictReader(src))
with open(sys.argv[2], encoding="utf-8", newline="") as dst:
    output_rows = sum(1 for _ in dst)

print(f"source_rows={source_rows} output_rows={output_rows}")
if source_rows != output_rows:
    raise SystemExit(1)
PY
```

Gere o checksum e somente então faça a troca:

```bash
sha256sum "$OUTPUT_CSV.tmp"
mv -- "$OUTPUT_CSV.tmp" "$OUTPUT_CSV"
```

## Registro e encerramento

Registre em `history/YYYY-MM-DD/registro.md`:

- caminhos ou nomes dos arquivos, sem dados pessoais;
- marcação usada;
- quantidade de linhas;
- quantidade de e-mails vazios e células com múltiplos e-mails;
- regra de arredondamento;
- resultado das validações;
- checksum SHA-256 do arquivo final;
- qualquer exceção ou decisão operacional.

Copie a versão usada deste procedimento para
`history/YYYY-MM-DD/procedimento_usado.md`. Remova arquivos temporários e
segredos ao final, mantendo backup somente quando houver necessidade de
rollback local.
