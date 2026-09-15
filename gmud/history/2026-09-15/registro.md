# Registro de execução - GMUD Jira para NocoDB

Data: 2026-09-15

## Contexto

Foi executado o procedimento de carga de GMUDs a partir do CSV exportado
do Jira em `gmud/.workspace/QueroQuitar!.csv`, com criação direta de
registros no PostgreSQL do NocoDB, auditoria em `public.nc_audit_v2`,
comentários em `public.nc_comments` e anexos nos campos de checklist e
evidências.

## Entradas

- CSV Jira: `gmud/.workspace/QueroQuitar!.csv`
- Checklists gerados: `gmud/.workspace/checklists/`
- Evidências zip geradas: `gmud/.workspace/evidencias_zip/`
- Estrutura S3 preparada: `gmud/.workspace/s3_upload/qq-it/`

## Decisões aplicadas

- O CSV completo do Jira não trouxe `Implantação`, `End date` ou
  `Target end` preenchidos. Foi usado `Updated` como referência de
  `GMUD."Implanta__o"`.
- O título da GMUD foi montado como `<Issue key> <Summary>`.
- O status real do NocoDB usado foi `4-Revisão`, não `4-Revisado`.
- Anderson Santos e Lucas Lyra foram removidos do lote. Eles não foram
  usados em auditorias nem comentários.
- Os anexos foram subidos manualmente para S3 porque o usuário AWS local
  do agente não tinha `s3:PutObject` no bucket `qq-it`.
- O token do MCP do NocoDB não autenticou os endpoints REST de upload.

## Execução

1. Confirmada conexão direta ao Postgres do NocoDB.
2. Confirmadas tabelas:
   - `pwof6z69xl5ubaz."GMUD"`
   - `public.nc_audit_v2`
   - `public.nc_comments`
3. Criado backup CSV local das três tabelas em `gmud/.workspace/backups/`.
4. Gerado SQL de carga das 39 GMUDs sem anexos.
5. Validado SQL em transação com `ROLLBACK`.
6. Aplicado SQL com `COMMIT`.
7. Detectada inconsistência de ordem temporal em parte dos históricos.
8. Gerado e aplicado reparo restrito aos registros novos.
9. Gerados checklists e zips de evidência.
10. Preparada estrutura local para upload S3.
11. Após confirmação de upload manual, preenchidos metadados dos anexos
    no banco e registradas auditorias de atualização.
12. Corrigidos os 10 zips PRD que inicialmente não haviam sido mapeados
    por causa de nomes em que a chave Jira ficava no fim e foi cortada
    no nome seguro do S3.
13. Após documentação do histórico e atualização do procedimento, as
    pastas temporárias `gmud/.workspace/` e `gmud/.secrets/` foram
    removidas da máquina local.

## Resultado final

- GMUDs criadas: 39
- IDs criados: 441 a 479
- Total da tabela GMUD após carga: 384
- Registros com `Checklist`: 39
- Registros com `Evidências PRD`: 33
- Auditorias de histórico:
  - `DATA_INSERT`: 39
  - `DATA_UPDATE` de status: 195
  - total de histórico inicial: 234
- Comentários criados: 125
- Auditorias de anexos criadas: 49
- Históricos válidos por sequência de status: 39 de 39
- Comentários/auditorias com Anderson Santos ou Lucas Lyra: 0

Sequência validada:

``` text
1-Aprovação
2-Aceito
3-Aplicado
4-Revisão
5-Fechado
```

## Artefatos SQL

- `gmud/.workspace/gmud_load_2026_no_attachments.sql`
- `gmud/.workspace/gmud_repair_2026_history.sql`
- `gmud/.workspace/gmud_attach_uploaded_s3.sql`
- `gmud/.workspace/gmud_attach_missing_prd.sql`

Também foram criadas versões `.rollback.sql` usadas para validação antes
do `COMMIT`. Esses artefatos estavam em `.workspace/`, pasta temporária
limpa ao final da execução.

## Validações finais

Consulta final de anexos:

``` text
rows = 39
checklist_filled = 39
prd_filled = 33
```

Consulta final de histórico:

``` text
valid_histories = 39
total_histories = 39
```

Foi validado via MCP/NocoDB que registros como `QQ-12769` e `QQ-12787`
retornam `Checklist` e `Evidências PRD` com `signedUrl`, confirmando que
o NocoDB reconheceu os metadados e os objetos enviados ao S3.

## Observações para próximas execuções

- Gerar nomes S3 que preservem a chave Jira no começo do arquivo sempre
  que possível. Isso evita falhas de mapeamento quando nomes longos são
  truncados.
- Sempre validar o SQL com `ROLLBACK` antes do `COMMIT`.
- Sempre validar a ordem cronológica dos status após a carga.
- Nunca limpar `nc_audit_v2` ou `nc_comments` apenas por `base_id`.
  Usar sempre `base_id + fk_model_id` e, quando possível, `row_id`.
- Para anexos, subir primeiro os arquivos no S3 e só depois gravar os
  metadados no banco.
- A confirmação definitiva dos anexos é o retorno do MCP/NocoDB com
  `signedUrl`.

## Procedimento copiado

Uma cópia do procedimento atualizado usado nesta execução está em:

``` text
gmud/history/2026-09-15/procedimento_usado.md
```
