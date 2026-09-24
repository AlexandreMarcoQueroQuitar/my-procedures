# Base avulsa Acredity

## Objetivo

Converter o arquivo de análise da Acredity para o formato de base avulsa lido
por `qq/pkg/campaign_engine/action/read_standalone_lead.go`, mantendo somente
as pessoas cuja margem disponível CLT tenha sido aprovada, isto é, seja maior
que zero.

O procedimento gera linhas sem cabeçalho no formato:

```text
CPF;acredity;aprovados_acredity_0826
```

As três posições são:

1. documento CPF;
2. credor/marcação lógica `acredity`;
3. marcação do lote.

Este formato não possui telefone, e-mail, flags de bypass nem coluna `data`.

## Comportamento no QQ

Antes de cada execução, confira o contrato no branch que será usado. Os
arquivos relevantes são:

- `pkg/campaign_engine/action/read_standalone_lead.go`;
- `pkg/campaign_engine/flow/loading_standalone_leads_file.go`;
- `pkg/campaign_engine/flow/processing_fast_track_items.go`;
- `pkg/campaign_engine/flow/processing_person_to_activation.go`;
- `pkg/campaign_engine/config/rules.go`.

No comportamento verificado em setembro de 2026:

- três colunas são suficientes para uma linha válida;
- o campo `creditor` é transportado com o lead;
- `read_standalone_lead` define `no_debt=true`;
- `bases_avulsas_email` e `bases_avulsas_sms` aceitam pessoas sem dívida;
- sem telefone ou e-mail no arquivo, nenhum contato novo é criado;
- a pessoa só é encaminhada para o canal em que já aparecer no cache de
  contato correspondente;
- uma pessoa sem contato elegível nos caches não será ativada pelo arquivo.

O valor `acredity` não cria um provider de contato nesta variação, pois isso
só ocorre quando a linha fornece telefone ou e-mail novo.

## Entrada

O arquivo observado foi:

```text
~/Downloads/acredity_marcacao/AN_QUERO_QUITAR 1.txt
```

Características esperadas:

- UTF-8, com ou sem BOM;
- delimitador `|`;
- cabeçalho;
- colunas obrigatórias `cpf` e `vl_margem_disponivel_clt`;
- margem numérica usando ponto como separador decimal.

As demais colunas não devem ser copiadas para a base avulsa. Não exponha CPF,
nome ou outros dados pessoais em logs; registre apenas contagens agregadas.

## Preparação segura

Crie as áreas locais, que não devem ser commitadas:

```bash
mkdir -p acredity-base-avulsa/.workspace acredity-base-avulsa/.secrets
```

Defina caminhos e marcação explicitamente:

```bash
SOURCE_FILE="$HOME/Downloads/acredity_marcacao/AN_QUERO_QUITAR 1.txt"
OUTPUT_FILE="$HOME/Downloads/acredity_marcacao/aprovados_acredity_0826_base_avulsa.csv"
MARK=aprovados_acredity_0826
```

Se o destino já existir, guarde uma cópia recuperável antes de substituí-lo:

```bash
cp -- "$OUTPUT_FILE" acredity-base-avulsa/.workspace/arquivo-anterior.csv
```

## Conversão e filtro

O filtro deve usar valor numérico, nunca comparação textual:

```text
Decimal(vl_margem_disponivel_clt) > 0
```

Margem vazia, zero ou negativa fica fora da saída. Um valor preenchido mas
inválido deve interromper o processamento, pois não é seguro inferir aprovação.

Execute a conversão para um arquivo temporário:

```bash
python3 - "$SOURCE_FILE" "$OUTPUT_FILE.tmp" "$MARK" <<'PY'
import csv
import sys
from decimal import Decimal, InvalidOperation
from pathlib import Path

source = Path(sys.argv[1])
output = Path(sys.argv[2])
mark = sys.argv[3]

if not mark:
    raise SystemExit("marcação vazia")
if any(c in mark for c in ";\r\n"):
    raise SystemExit("marcação contém delimitador inválido")

def valid_cpf(document):
    if len(document) != 11 or not document.isdigit() or document == document[0] * 11:
        return False
    for size in (9, 10):
        total = sum(int(document[i]) * (size + 1 - i) for i in range(size))
        digit = (total * 10) % 11
        if digit == 10:
            digit = 0
        if digit != int(document[size]):
            return False
    return True

source_rows = 0
approved_rows = 0
seen = set()

with source.open("r", encoding="utf-8-sig", newline="") as src, output.open(
    "w", encoding="utf-8", newline=""
) as dst:
    reader = csv.DictReader(src, delimiter="|")
    required = {"cpf", "vl_margem_disponivel_clt"}
    if not reader.fieldnames or not required.issubset(reader.fieldnames):
        raise SystemExit(
            "cabeçalho inválido; esperadas cpf e vl_margem_disponivel_clt"
        )

    for row in reader:
        source_rows += 1
        raw_margin = (row.get("vl_margem_disponivel_clt") or "").strip()
        if not raw_margin:
            continue
        try:
            margin = Decimal(raw_margin)
        except InvalidOperation:
            raise SystemExit(f"margem inválida na linha de dados {source_rows}")
        if margin <= 0:
            continue

        cpf = "".join(c for c in (row.get("cpf") or "") if c.isdigit())
        if not valid_cpf(cpf):
            raise SystemExit(f"CPF inválido na linha de dados {source_rows}")
        if cpf in seen:
            raise SystemExit(f"CPF aprovado duplicado na linha de dados {source_rows}")
        seen.add(cpf)

        dst.write(f"{cpf};acredity;{mark}\n")
        approved_rows += 1

print(f"source_rows={source_rows} approved_rows={approved_rows}")
PY
```

O script não escreve cabeçalho e não inclui a margem no resultado; ela é usada
exclusivamente como critério de seleção.

## Validação antes da substituição

Valide a estrutura sem imprimir documentos:

```bash
python3 - "$OUTPUT_FILE.tmp" "$MARK" <<'PY'
import sys
from pathlib import Path

path = Path(sys.argv[1])
expected_mark = sys.argv[2]
rows = bad_width = bad_fixed = bad_document = duplicates = 0
seen = set()

with path.open("r", encoding="utf-8", newline="") as src:
    for line in src:
        rows += 1
        columns = line.rstrip("\r\n").split(";")
        if len(columns) != 3:
            bad_width += 1
            continue
        document, creditor, mark = columns
        if creditor != "acredity" or mark != expected_mark:
            bad_fixed += 1
        if len(document) != 11 or not document.isdigit():
            bad_document += 1
        if document in seen:
            duplicates += 1
        seen.add(document)

print(
    f"rows={rows} bad_width={bad_width} bad_fixed={bad_fixed} "
    f"bad_document={bad_document} duplicates={duplicates}"
)
if any((bad_width, bad_fixed, bad_document, duplicates)):
    raise SystemExit(1)
PY
```

Gere o checksum e somente então faça a troca:

```bash
sha256sum "$OUTPUT_FILE.tmp"
mv -- "$OUTPUT_FILE.tmp" "$OUTPUT_FILE"
```

## Registro e encerramento

Registre em `history/YYYY-MM-DD/registro.md`:

- nome da origem e do destino, sem conteúdo pessoal;
- marcação usada;
- total de linhas da origem;
- quantidade selecionada por margem maior que zero;
- contagem de margens vazias, zero, negativas ou inválidas;
- contagem de CPFs inválidos ou duplicados;
- resultado das validações;
- checksum SHA-256 final;
- exceções e decisões operacionais.

Copie a versão usada deste procedimento para
`history/YYYY-MM-DD/procedimento_usado.md`. Remova arquivos temporários e
segredos ao final, mantendo backup somente quando necessário para rollback.
