# Base avulsa Noverde

## Objetivo

Converter um CSV de leads da Noverde para o formato de base avulsa lido por
`qq/pkg/campaign_engine/action/read_standalone_lead.go`, usando o e-mail do
arquivo de origem e levando `amount` e `max_period` no JSON da coluna de dados.

O procedimento gera linhas sem cabeçalho no formato:

```text
documento;noverde;marcacao;;email;0;1;{"gc_credit_amount": "R$ 1.234,56", "gc_credit_inst": "15"}
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
- conter, no mínimo, as colunas `document`, `email`, `amount` e `max_period`;
- preservar documentos como texto, sem notação científica;
- ter `amount` numérico usando ponto como separador decimal;
- ter `max_period` como inteiro positivo, que será preservado como texto em
  `gc_credit_inst`.

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
- copia `max_period` para `gc_credit_inst` como texto;
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

def format_period(raw):
    value = (raw or "").strip()
    if not value.isascii() or not value.isdecimal() or int(value) <= 0:
        raise ValueError("max_period vazio ou inválido")
    return value

rows = 0
empty_email = 0
multiple_email = 0

with source.open("r", encoding="utf-8-sig", newline="") as src, output.open(
    "w", encoding="utf-8", newline=""
) as dst:
    reader = csv.DictReader(src)
    required = {"document", "email", "amount", "max_period"}
    if not reader.fieldnames or not required.issubset(reader.fieldnames):
        raise SystemExit(
            "cabeçalho inválido; esperadas as colunas document, email, amount e max_period"
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
            {
                "gc_credit_amount": format_brl(row.get("amount")),
                "gc_credit_inst": format_period(row.get("max_period")),
            },
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

Se qualquer `amount` ou `max_period` estiver ausente ou inválido, a execução
deve falhar. Não preencha valores por inferência.

## Validação antes da substituição

Valide todas as linhas e o JSON sem imprimir dados pessoais:

```bash
python3 - "$OUTPUT_CSV.tmp" <<'PY'
import json
import sys
from pathlib import Path

path = Path(sys.argv[1])
rows = bad_width = bad_fixed = bad_json = bad_currency = bad_period = 0

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

        value = obj.get("gc_credit_amount")
        if not isinstance(value, str) or not value.startswith("R$ "):
            bad_currency += 1
        period = obj.get("gc_credit_inst")
        if (
            set(obj) != {"gc_credit_amount", "gc_credit_inst"}
            or not isinstance(period, str)
            or not period.isascii()
            or not period.isdecimal()
            or int(period) <= 0
        ):
            bad_period += 1

print(
    f"rows={rows} bad_width={bad_width} bad_fixed={bad_fixed} "
    f"bad_json={bad_json} bad_currency={bad_currency} bad_period={bad_period}"
)

if any((bad_width, bad_fixed, bad_json, bad_currency, bad_period)):
    raise SystemExit(1)
PY
```

Compare também os campos gerados com a origem, linha a linha:

```bash
python3 - "$SOURCE_CSV" "$OUTPUT_CSV.tmp" <<'PY'
import csv
import json
import sys
from decimal import Decimal, ROUND_HALF_UP
from itertools import zip_longest

with open(sys.argv[1], encoding="utf-8-sig", newline="") as src:
    with open(sys.argv[2], encoding="utf-8", newline="") as dst:
        rows = 0
        for rows, (original, converted) in enumerate(
            zip_longest(csv.DictReader(src), dst), 1
        ):
            if original is None or converted is None:
                raise SystemExit(f"quantidade de linhas diferente na posição {rows}")
            columns = converted.rstrip("\r\n").split(";", 7)
            if len(columns) != 8:
                raise SystemExit(f"largura inválida na posição {rows}")
            value = Decimal(original["amount"].strip()).quantize(
                Decimal("0.01"), rounding=ROUND_HALF_UP
            )
            currency = f"R$ {value:,.2f}".replace(",", "X").replace(
                ".", ","
            ).replace("X", ".")
            expected_email = (original["email"] or "").replace("\r", "").replace(
                "\n", ""
            ).split(";", 1)[0].strip()
            data = json.loads(columns[7])
            if (
                columns[0] != original["document"].strip()
                or columns[4] != expected_email
                or data.get("gc_credit_amount") != currency
                or data.get("gc_credit_inst") != original["max_period"].strip()
            ):
                raise SystemExit(f"dados divergentes na posição {rows}")

print(f"source_output_rows_matched={rows}")
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
- validação de `max_period` e dos campos `gc_credit_amount` e
  `gc_credit_inst`;
- regra de arredondamento;
- resultado das validações;
- checksum SHA-256 do arquivo final;
- qualquer exceção ou decisão operacional.

Copie a versão usada deste procedimento para
`history/YYYY-MM-DD/procedimento_usado.md`. Remova arquivos temporários e
segredos ao final, mantendo backup somente quando houver necessidade de
rollback local.
