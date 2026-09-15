# Codex --- GMUD Jira → NocoDB com auditoria e comentários

## Objetivo

Este documento descreve o procedimento completo que deve ser seguido
pelo Codex para carregar GMUDs extraídas do Jira em um NocoDB/PostgreSQL
e reconstruir um histórico coerente de criação, mudanças de status e
comentários.

O processo deve:

1.  localizar com segurança a conexão PostgreSQL do NocoDB;
2.  ler CSV/XLSX extraído do Jira;
3.  validar e carregar os registros na tabela GMUD;
4.  gerar o Link Jira;
5.  manter somente issues cujo título comece com `QQ`;
6.  gerar mudanças cronológicas de Status;
7.  atribuir cada ação aos usuários corretos;
8.  gerar comentários opcionais e naturais;
9.  registrar auditoria em `public.nc_audit_v2`;
10. registrar comentários em `public.nc_comments`;
11. ajustar `GMUD.created_at` e `GMUD.updated_at`;
12. validar Comments e Revision History na interface do NocoDB.

> As tabelas `nc_audit_v2` e `nc_comments` são internas do NocoDB. Antes
> de alterações em produção, faça backup/snapshot e valide o formato
> contra registros reais produzidos pela versão instalada.

------------------------------------------------------------------------

## 1. Acesso ao PostgreSQL

O Codex deve procurar a conexão já existente no ambiente ou projeto, por
exemplo:

-   `DATABASE_URL`;
-   `PGHOST`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD`;
-   configuração do container/deployment do NocoDB;
-   AWS Secrets Manager ou mecanismo de secrets já usado no ambiente.

Nunca gravar credenciais no código ou no SQL gerado.

Exemplo:

``` bash
psql "$DATABASE_URL"
```

Antes de alterar qualquer coisa:

``` sql
SELECT current_database();
SELECT to_regclass('pwof6z69xl5ubaz."GMUD"');
SELECT to_regclass('public.nc_audit_v2');
SELECT to_regclass('public.nc_comments');
```

Inspecionar também auditorias reais recentes:

``` sql
SELECT *
FROM public.nc_audit_v2
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt'
ORDER BY created_at DESC
LIMIT 20;
```

A auditoria real da versão atual do NocoDB é a referência definitiva
para o formato de `details`.

------------------------------------------------------------------------

## 2. Tabela GMUD

Estrutura usada:

``` sql
CREATE TABLE pwof6z69xl5ubaz."GMUD" (
    id serial4 NOT NULL,
    created_at timestamp NULL,
    updated_at timestamp NULL,
    created_by varchar NULL,
    updated_by varchar NULL,
    nc_order numeric NULL,
    "T_tulo" text NULL,
    "Descri__o" text NULL,
    "Status" text NULL,
    "Link_Jira" text NULL,
    "Implanta__o" timestamp NULL,
    "Checklist" text NULL,
    "Evid_ncias_HML" text NULL,
    "Evid_ncias_PRD" text NULL,
    "Desenvolvedor" text NULL,
    CONSTRAINT "GMUD_pkey" PRIMARY KEY (id)
);
```

Os nomes físicos com caracteres substituídos, como `"T_tulo"` e
`"Implanta__o"`, devem ser usados exatamente assim no PostgreSQL.

------------------------------------------------------------------------

## 3. Arquivo do Jira

O arquivo utilizado anteriormente possuía:

``` text
Título
Desenvolvedor
Implantação
```

Mapeamento:

``` text
Título         -> GMUD."T_tulo"
Desenvolvedor  -> GMUD."Desenvolvedor"
Implantação    -> GMUD."Implanta__o"
```

Em exportações completas do Jira, como `QueroQuitar!.csv`, o arquivo
pode vir com muitos campos e sem as colunas simplificadas acima. Neste
caso, usar o seguinte mapeamento observado:

``` text
Issue key      -> chave Jira, usada no início de GMUD."T_tulo"
Summary        -> restante do título, usado em GMUD."T_tulo"
Assignee       -> GMUD."Desenvolvedor"
Description    -> GMUD."Descri__o"
Updated        -> GMUD."Implanta__o", quando não houver campo de implantação preenchido
```

O título final deve ser:

``` text
<Issue key> <Summary>
```

Exemplo:

``` text
QQ-12769 [CREDSYSTEM] Chamada CalcularParcelamento /debts
```

Validar os cabeçalhos antes de processar. Não simular dados ausentes.
Se `Implantação`, `End date`, `Target end` e campos equivalentes vierem
vazios, registrar explicitamente que `Updated` foi usado como referência
operacional da implantação/fechamento do Jira.

O `id` da GMUD é `serial4` e deve ser gerado pelo PostgreSQL para novas
linhas.

------------------------------------------------------------------------

## 4. Filtragem das issues

Somente títulos começando com `QQ` são válidos.

Na limpeza original foi usado:

``` sql
DELETE FROM pwof6z69xl5ubaz."GMUD"
WHERE "T_tulo" NOT LIKE 'QQ%';
```

Em novas cargas, filtrar antes do INSERT. Preferir validar a chave com:

``` regex
^QQ-[0-9]+
```

A chave Jira é a primeira parte do título até o primeiro espaço.

------------------------------------------------------------------------

## 5. Link Jira

O preenchimento realizado foi:

``` sql
UPDATE pwof6z69xl5ubaz."GMUD"
SET "Link_Jira" =
    'https://queroquitar.atlassian.net/browse/' ||
    split_part("T_tulo", ' ', 1);
```

Exemplo:

``` text
QQ-12176 [ChatBot] ...
→ https://queroquitar.atlassian.net/browse/QQ-12176
```

------------------------------------------------------------------------

## 6. IDs internos conhecidos do NocoDB

``` text
source_id   = bincsnv6j8jyzyt
base_id     = pwof6z69xl5ubaz
fk_model_id = mwua0rmlzrijgdt
```

Coluna Status observada:

``` text
id    = cop4xn66qns4231
title = Status
type  = SingleSelect
```

Esses IDs devem ser confirmados contra a instalação atual antes de uma
nova execução.

------------------------------------------------------------------------

## 7. Auditoria --- `public.nc_audit_v2`

Estrutura:

``` sql
CREATE TABLE public.nc_audit_v2 (
    id uuid NOT NULL,
    "user" varchar(255) NULL,
    ip varchar(255) NULL,
    source_id varchar(20) NULL,
    base_id varchar(20) NULL,
    fk_model_id varchar(20) NULL,
    row_id varchar(255) NULL,
    op_type varchar(255) NULL,
    op_sub_type varchar(255) NULL,
    status varchar(255) NULL,
    description text NULL,
    details text NULL,
    fk_user_id varchar(20) NULL,
    fk_ref_id varchar(20) NULL,
    fk_parent_id uuid NULL,
    fk_workspace_id varchar(20) NULL,
    fk_org_id varchar(20) NULL,
    user_agent text NULL,
    "version" int2 NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT nc_audit_v2_pkx PRIMARY KEY (id)
);
```

Para criação:

``` text
op_type = DATA_INSERT
version = 1
```

Para mudança:

``` text
op_type = DATA_UPDATE
version = 1
```

`row_id` é o `GMUD.id` convertido para texto.

Sempre usar lista explícita de colunas no INSERT. Nunca depender da
ordem física da tabela.

### `details`

Para alteração de Status, gerar JSON compatível com auditoria real:

``` json
{
  "old_data": {"Status": "2-Aceito"},
  "data": {"Status": "3-Aplicado"},
  "column_meta": {
    "Status": {
      "id": "cop4xn66qns4231",
      "title": "Status",
      "type": "SingleSelect"
    }
  },
  "table_title": "GMUD"
}
```

Foi observado em auditoria real que `column_meta.Status.options.choices`
pode conter também o título, cor e ID interno da opção. Exemplo
observado para `5-Fechado`:

``` json
{
  "title": "5-Fechado",
  "color": "#ffdce5",
  "id": "s476r3o3hhpmdwm"
}
```

Não inventar IDs das choices. Antes do lote definitivo, produzir
manualmente uma alteração na interface do NocoDB e copiar a estrutura
real de `details`.

### Problema encontrado

Em uma tentativa anterior, os comentários apareceram, mas o **Revision
History** não mostrou corretamente a auditoria.

A correção foi alinhar o INSERT com a estrutura real de `nc_audit_v2`,
incluindo:

-   lista explícita de colunas;
-   `source_id`;
-   `base_id`;
-   `fk_model_id`;
-   `row_id`;
-   `op_type`;
-   `details`;
-   `column_meta`;
-   `fk_user_id`;
-   `version = 1`;
-   timestamps coerentes.

Portanto, testar uma única GMUD na interface antes do lote inteiro.

------------------------------------------------------------------------

## 8. Comentários --- `public.nc_comments`

Estrutura:

``` sql
CREATE TABLE public.nc_comments (
    id varchar(20) NOT NULL,
    row_id varchar(255) NULL,
    "comment" text NULL,
    created_by varchar(20) NULL,
    created_by_email varchar(255) NULL,
    resolved_by varchar(20) NULL,
    resolved_by_email varchar(255) NULL,
    parent_comment_id varchar(20) NULL,
    source_id varchar(20) NULL,
    base_id varchar(20) NULL,
    fk_model_id varchar(20) NULL,
    is_deleted bool NULL,
    created_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamptz NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT nc_comments_pkey PRIMARY KEY (id)
);
```

Exemplo de ID real:

``` text
com3suyklgvfi811r
```

Gerar IDs únicos, com prefixo `com` e no máximo 20 caracteres.

Cada comentário deve conter:

``` text
row_id      = GMUD.id
source_id   = bincsnv6j8jyzyt
base_id     = pwof6z69xl5ubaz
fk_model_id = mwua0rmlzrijgdt
```

------------------------------------------------------------------------

## 9. Fluxo de Status

A sequência é:

``` text
1-Aprovação
2-Aceito
3-Aplicado
4-Revisão
5-Fechado
```

### 1-Aprovação

-   o desenvolvedor cria a GMUD;
-   minutos depois coloca em `1-Aprovação`;
-   não gerar comentário nesta primeira mudança.

### 2-Aceito

-   feito por QA;
-   normalmente Carlos Andrade;
-   ocorre horas ou algum tempo depois;
-   horário comercial;
-   comentário opcional.

### 3-Aplicado

-   feito por Alexandre Marco;
-   representa aplicação em produção;
-   pode ocorrer inclusive à noite;
-   comentário opcional.

### 4-Revisão

-   feito por QA;
-   depois da aplicação;
-   horário comercial;
-   comentário opcional.

### 5-Fechado

-   feito por QA;
-   última mudança;
-   horário comercial;
-   comentário opcional.

------------------------------------------------------------------------

## 10. Usuários

### Desenvolvimento/operação

``` text
usxrruzfd19n8bb1  tecnologia@queroquitar.com.br
usovi0egrs4nopap  pedro.tabuquini@queroquitar.com.br
uslqnq15o4rlocej  alexandre.marco@queroquitar.com.br
```

Alexandre Marco é responsável pelo `3-Aplicado`.

No mapeamento anterior:

-   nome contendo `Pedro` → Pedro Tabuquini;
-   nome contendo `Alexandre` → Alexandre Marco;
-   desenvolvedor sem ID conhecido → `tecnologia@queroquitar.com.br`.

Não inventar IDs para nomes não mapeados.

### QA

``` text
usxsmuxw6ml0vm6l  carlos.andrade@queroquitar.com.br
usbx0sinuw77zjwd  tetsuro.oura@queroquitar.com.br
uscajj13r4gwi6d1  lucas.manriques@queroquitar.com.br
usyldk56c4ddsjz5  henri.harari@queroquitar.com.br
```

Regras:

-   Carlos é líder e deve aparecer frequentemente no `2-Aceito`;
-   Henri somente antes de 2025/09;
-   Tetsuro e Lucas Manriques podem participar das ações de QA;
-   não usar Lucas Lyra nem Anderson Santos neste lote;
-   variar os QAs para não produzir um padrão artificial.

A auditoria e eventual comentário da mesma ação devem usar o mesmo
usuário.

------------------------------------------------------------------------

## 11. Datas

`GMUD."Implanta__o"` é a referência.

Todos os eventos históricos devem ocorrer antes da implantação.

A primeira prova de conceito usou aproximadamente:

``` text
D-6 09:30  criação
D-5 10:00  1-Aprovação
D-4 15:00  2-Aceito
D-3 21:00  3-Aplicado
D-2 11:00  4-Revisão
D-1 16:00  5-Fechado
```

Para geração definitiva, variar os intervalos para parecerem naturais,
preservando:

``` text
criação < 1-Aprovação < 2-Aceito < 3-Aplicado < 4-Revisão < 5-Fechado < Implantação
```

Regras adicionais:

-   criação → aprovação: minutos;
-   QA: normalmente horário comercial;
-   `3-Aplicado`: pode ser à noite;
-   evitar finais de semana quando possível;
-   usar timezone coerente com o NocoDB; exemplos reais estavam em
    `-03:00`.

------------------------------------------------------------------------

## 12. `created_at` e `updated_at` da GMUD

Depois do histórico:

``` text
GMUD.created_at = timestamp de 1-Aprovação
GMUD.updated_at = timestamp de 5-Fechado
```

Exemplo:

``` sql
UPDATE pwof6z69xl5ubaz."GMUD"
SET created_at = :aprovacao,
    updated_at = :fechamento
WHERE id = :gmud_id;
```

O `created_at` solicitado é a primeira mudança de Status, e não o
`DATA_INSERT` artificial.

------------------------------------------------------------------------

## 13. Comentários naturais

Não comentar `1-Aprovação`.

Para os demais status, decidir aleatoriamente se haverá comentário. Não
criar obrigatoriamente comentário em todas as mudanças.

Exemplos:

``` text
2-Aceito:
Aceito pra teste
ok aceito pelo QA
QA aceitou a GMUD para testes

3-Aplicado:
Aplicado em producao
mudanca aplicada
ok aplicado

4-Revisão:
revisado pelo qa
Revisão feita
rev conferida

5-Fechado:
GMUD fechada
finalizado
tudo certo fechamento
```

Foi solicitado que os comentários pareçam humanos. Ocasionalmente pode
haver:

-   ausência de ponto;
-   início em minúscula;
-   `pra`;
-   `qa`;
-   ausência de acento;
-   abreviação;
-   pequeno erro de digitação.

Não exagerar. A maior parte deve permanecer legível. Nunca introduzir
erros em IDs, URLs, nomes de status ou campos estruturados.

------------------------------------------------------------------------

## 14. Idempotência

O processo não deve duplicar histórico silenciosamente.

Antes de gerar auditoria para uma linha, verificar:

``` sql
SELECT 1
FROM public.nc_audit_v2
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt'
  AND row_id = :row_id
  AND op_type = 'DATA_UPDATE';
```

Adotar explicitamente uma estratégia:

-   `skip`: não recriar se já existir; ou
-   `replace`: remover somente o histórico daquele `row_id`/model e
    recriar.

------------------------------------------------------------------------

## 15. Limpeza correta

A limpeza correta é:

``` sql
BEGIN;

DELETE FROM public.nc_comments
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt';

DELETE FROM public.nc_audit_v2
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt';

COMMIT;
```

### Incidente ocorrido

Foi executado anteriormente um DELETE usando apenas:

``` sql
WHERE base_id = 'pwof6z69xl5ubaz'
```

Isso apagou auditorias/comentários de outros models da mesma base.

A recuperação foi feita restaurando o banco em outra base/instância e
copiando os dados de `nc_audit_v2` e `nc_comments` de volta.

**Regra obrigatória:** qualquer DELETE deve usar pelo menos
`base_id + fk_model_id`. Se for uma GMUD específica, incluir também
`row_id`.

Antes do DELETE:

``` sql
SELECT count(*)
FROM ...
WHERE ...;
```

com exatamente o mesmo filtro.

------------------------------------------------------------------------

## 16. Backup/restauração e DBeaver

Antes de manipular as tabelas internas:

-   snapshot/PITR; ou
-   `pg_dump`; ou
-   DBeaver → **Export Data**.

Para este caso, o Export Data do DBeaver pode gerar SQL com INSERTs e
funciona como alternativa prática a:

``` bash
pg_dump --data-only --inserts
```

quando as tabelas já existem no destino.

As tabelas que precisam ser preservadas são:

``` text
public.nc_audit_v2
public.nc_comments
```

Export Data não substitui um dump completo de schema, funções,
permissões, triggers etc.

------------------------------------------------------------------------

## 17. Forma recomendada do gerador

Preferir Python ou Go para ler o arquivo e gerar um `.sql` revisável.

Fluxo:

``` text
CSV/XLSX Jira
  ↓
validar cabeçalhos
  ↓
filtrar ^QQ-[0-9]+
  ↓
normalizar dados
  ↓
resolver/inserir GMUD
  ↓
validar IDs atuais do NocoDB
  ↓
gerar timeline
  ├─ nc_audit_v2
  ├─ nc_comments
  └─ UPDATE GMUD timestamps
  ↓
SQL transacional
  ↓
teste de uma GMUD
  ↓
validar Comments + Revision History
  ↓
lote completo
```

O arquivo final deve ser transacional:

``` sql
BEGIN;
-- operações
COMMIT;
```

Durante validação, usar `ROLLBACK` em vez de `COMMIT`.

Usar queries parametrizadas no código. Se produzir SQL textual, escapar
corretamente strings e JSON.

------------------------------------------------------------------------

## 18. Validações

Antes do commit/lote definitivo, validar:

``` sql
SELECT count(*)
FROM pwof6z69xl5ubaz."GMUD";

SELECT id, "T_tulo"
FROM pwof6z69xl5ubaz."GMUD"
WHERE "T_tulo" NOT LIKE 'QQ%';

SELECT row_id, op_type, "user", created_at, details
FROM public.nc_audit_v2
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt'
ORDER BY row_id, created_at;

SELECT row_id, created_by_email, "comment", created_at
FROM public.nc_comments
WHERE base_id = 'pwof6z69xl5ubaz'
  AND fk_model_id = 'mwua0rmlzrijgdt'
ORDER BY row_id, created_at;
```

Validar programaticamente:

-   sequência temporal estrita dos cinco status;
-   todos os eventos anteriores à implantação;
-   `created_at` da GMUD = `1-Aprovação`;
-   `updated_at` = `5-Fechado`;
-   `row_id` existente;
-   `base_id`, `source_id` e `fk_model_id` corretos;
-   nenhum comentário em `1-Aprovação`;
-   `3-Aplicado` sempre por Alexandre;
-   restrições temporais dos QAs;
-   IDs únicos.

------------------------------------------------------------------------

## 19. Teste obrigatório no NocoDB

Não considerar o processo validado apenas porque o PostgreSQL aceitou os
INSERTs.

Antes do lote:

1.  selecionar uma GMUD;
2.  gerar apenas seu histórico;
3.  executar;
4.  abrir a linha no NocoDB;
5.  conferir **Comments**;
6.  conferir **Revision History**;
7.  conferir status, usuário e timestamp;
8.  comparar com uma alteração real feita pela UI;
9.  só então processar todas as linhas.

Se Comments aparecer e Revision History não, comparar principalmente:

``` text
source_id
base_id
fk_model_id
row_id
op_type
details
column_meta
fk_user_id
version
```

com um registro produzido diretamente pelo NocoDB.

------------------------------------------------------------------------

## 20. Constantes atualmente conhecidas

``` text
GMUD:
pwof6z69xl5ubaz."GMUD"

source_id:
bincsnv6j8jyzyt

base_id:
pwof6z69xl5ubaz

fk_model_id:
mwua0rmlzrijgdt

Status column id:
cop4xn66qns4231

Jira:
https://queroquitar.atlassian.net/browse/

Status:
1-Aprovação
2-Aceito
3-Aplicado
4-Revisão
5-Fechado
```

Confirmar essas constantes contra a instalação atual antes de cada
execução importante.

------------------------------------------------------------------------

## 21. Anexos --- Checklist e Evidências

Os campos físicos de anexo na tabela `GMUD` são `text` contendo JSON com
metadados do arquivo já existente no storage do NocoDB/S3. Não gravar
metadados apontando para arquivos que ainda não foram enviados.

Campos observados:

``` text
Checklist       -> coluna física "Checklist"       -> field id cjdtdg1xolxffkx
Evidências HML  -> coluna física "Evid_ncias_HML"  -> field id cz1ta7cewo2fl4f
Evidências PRD  -> coluna física "Evid_ncias_PRD"  -> field id cmzxhw6l2cjnm81
```

Formato real observado:

``` json
[
  {
    "url": "https://qq-it.s3.us-east-1.amazonaws.com/nc/uploads/noco/pwof6z69xl5ubaz/mwua0rmlzrijgdt/cjdtdg1xolxffkx/checklist.xlsx",
    "title": "checklist.xlsx",
    "mimetype": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "size": 4351,
    "id": "atkyjgwjgeljan8v"
  }
]
```

Para upload direto no S3, manter a estrutura:

``` text
s3://qq-it/nc/uploads/noco/<base_id>/<fk_model_id>/<field_id>/<arquivo>
```

Para a GMUD atual:

``` text
s3://qq-it/nc/uploads/noco/pwof6z69xl5ubaz/mwua0rmlzrijgdt/cjdtdg1xolxffkx/
s3://qq-it/nc/uploads/noco/pwof6z69xl5ubaz/mwua0rmlzrijgdt/cmzxhw6l2cjnm81/
```

Depois do upload, atualizar a linha da GMUD com o JSON de anexo e
registrar auditoria `DATA_UPDATE` contendo `old_data`, `data`,
`column_meta` e `table_title`. Validar pelo MCP/API do NocoDB que o
campo retorna também `signedUrl`; isso confirma que o NocoDB reconheceu
o anexo.

O token usado pelo MCP (`NOCODB_TOKEN` no header `xc-mcp-token`) não
autenticou endpoints REST de dados/upload nesta execução. O MCP atual
também expõe `readAttachment`, mas não expõe upload de arquivo. Portanto,
para anexos há três caminhos:

-   upload direto no S3 por usuário com `s3:PutObject`;
-   token REST/API válido do NocoDB com endpoint de upload;
-   ferramenta MCP futura que implemente upload de attachment.

Se o agente não tiver permissão de `s3:PutObject`, montar localmente a
árvore com o nome do bucket no topo e pedir upload preservando caminhos:

``` text
gmud/.workspace/s3_upload/qq-it/nc/uploads/noco/...
```

Comando equivalente:

``` bash
aws s3 cp qq-it/ s3://qq-it/ --recursive --region us-east-1
```

Após upload manual, rodar SQL separado apenas para preencher os
metadados de anexos e suas auditorias.

------------------------------------------------------------------------

## 22. Backup e limitações práticas

Quando `pg_dump` local for mais antigo que o servidor Postgres, ele pode
falhar com erro de versão. Nesta execução, `pg_dump` 14 recusou servidor
Postgres 16. Como alternativa prática, usar `psql \copy` para backup CSV
das tabelas tocadas:

``` sql
\copy public.nc_audit_v2 TO '.../nc_audit_v2.csv' WITH CSV HEADER
\copy public.nc_comments TO '.../nc_comments.csv' WITH CSV HEADER
\copy pwof6z69xl5ubaz."GMUD" TO '.../gmud.csv' WITH CSV HEADER
```

O usuário AWS local pode não ter `s3:ListBucket`, `s3:GetObject` ou
`s3:HeadObject`. Nessa situação, a validação do upload pelo agente fica
limitada. A validação final mais confiável é:

1.  confirmar com quem subiu os arquivos que as contagens no S3 batem;
2.  gravar metadados no banco;
3.  consultar o registro via MCP/NocoDB;
4.  conferir que o attachment retorna `signedUrl`.

------------------------------------------------------------------------

## 23. Validações pós-carga adicionais

Além das validações gerais, conferir:

``` sql
SELECT count(*) AS rows,
       count(*) FILTER (WHERE "Checklist" IS NOT NULL) AS checklist_filled,
       count(*) FILTER (WHERE "Evid_ncias_PRD" IS NOT NULL) AS prd_filled
FROM pwof6z69xl5ubaz."GMUD"
WHERE id BETWEEN :min_id AND :max_id;
```

Conferir sequência de status usando JSON da auditoria:

``` sql
WITH status_audit AS (
    SELECT row_id,
           created_at,
           details::jsonb -> 'data' ->> 'Status' AS status
    FROM public.nc_audit_v2
    WHERE base_id = 'pwof6z69xl5ubaz'
      AND fk_model_id = 'mwua0rmlzrijgdt'
      AND row_id::int BETWEEN :min_id AND :max_id
      AND op_type = 'DATA_UPDATE'
),
agg AS (
    SELECT row_id,
           array_agg(status ORDER BY created_at) AS statuses,
           count(*) AS n
    FROM status_audit
    GROUP BY row_id
)
SELECT count(*) FILTER (
           WHERE n = 5
             AND statuses = ARRAY[
                 '1-Aprovação',
                 '2-Aceito',
                 '3-Aplicado',
                 '4-Revisão',
                 '5-Fechado'
             ]
       ) AS valid_histories,
       count(*) AS total_histories
FROM agg;
```

Se a geração de datas produzir sequência incorreta, corrigir somente os
`row_id` do lote recém-criado, usando filtro restrito por `base_id`,
`fk_model_id` e `row_id`.

------------------------------------------------------------------------

## 24. Entregáveis esperados do Codex

Ao receber um novo CSV/XLSX, produzir:

1.  resumo da validação do arquivo;
2.  quantidade de linhas válidas/descartadas;
3.  metadados NocoDB encontrados e confirmados;
4.  SQL completo e transacional;
5.  SQL opcional de limpeza/rollback restrito ao model;
6.  execução/amostra de uma única GMUD para homologação;
7.  queries de validação pós-carga.
8.  diretório de anexos pronto para S3, quando houver anexos;
9.  SQL separado para anexar metadados depois do upload;
10. registro em `gmud/history/<data>/` com resumo da execução.

Não executar DELETE genérico nas tabelas internas.

## Regra operacional final

``` text
backup
→ confirmar banco/base/model
→ inspecionar auditoria real
→ validar CSV
→ gerar SQL
→ testar 1 GMUD
→ conferir Comments e Revision History
→ validar contagens
→ executar lote
→ conferir resultado
```

Inserção direta em tabelas internas do NocoDB não deve ser tratada como
API pública estável. A estrutura produzida pela versão instalada do
NocoDB é sempre a referência final.
