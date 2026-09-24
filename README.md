# my-procedures

Repositório de procedimentos operacionais reutilizáveis da QueroQuitar.

Cada procedimento deve ficar em uma pasta própria na raiz do repositório.

Procedimentos disponíveis:

- `acredity-base-avulsa/`: filtra margem aprovada e formata CPFs da Acredity
  para base avulsa do `campaign_engine`;
- `gmud/`: carga de GMUDs do Jira para o NocoDB;
- `noverde-base-avulsa/`: formatação de CSV da Noverde para base avulsa do
  `campaign_engine`, incluindo e-mail e valor de crédito no JSON adicional.

## Estrutura Padrão

Cada pasta de procedimento deve seguir este padrão:

```text
<procedimento>/
  README.md ou CODEX_<NOME>.md
  history/
    YYYY-MM-DD/
      registro.md
      procedimento_usado.md
  .workspace/
  .secrets/
```

`README.md` ou `CODEX_<NOME>.md` descreve o procedimento vivo, isto é, a
versão que deve ser usada e aprimorada nas próximas execuções.

`history/YYYY-MM-DD/registro.md` guarda o resumo de uma execução real:
entradas usadas, decisões tomadas, comandos ou artefatos relevantes,
validações, resultado final e pendências.

`history/YYYY-MM-DD/procedimento_usado.md` guarda uma cópia do
procedimento usado naquela data, já com os aprendizados incorporados
quando fizer sentido. Essa cópia serve para auditoria e reprodução.

`.workspace/` é área temporária de execução. Pode receber CSVs, SQLs,
backups locais, arquivos gerados, pacotes para upload e outros artefatos
operacionais. Ela não deve ser commitada e deve ser limpa ao final da
execução quando não houver necessidade local de retenção.

`.secrets/` é área temporária para credenciais locais específicas de uma
execução. Ela não deve ser commitada e não deve permanecer na máquina
depois do uso. Preferir sempre o cofre QQ ou mecanismos oficiais de
segredo quando disponíveis.

## Regras de Execução

- Ler o procedimento antes de executar.
- Confirmar ambiente, credenciais e alvos antes de qualquer escrita.
- Fazer backup ou exportação das tabelas/recursos impactados quando a
  operação tocar dados persistentes.
- Validar scripts ou SQLs com `ROLLBACK`, dry-run ou equivalente antes
  do `COMMIT`.
- Usar filtros restritos em limpezas e reparos. Nunca apagar dados por
  filtros amplos quando houver identificadores de lote, model, tabela ou
  linha disponíveis.
- Registrar no `history/` tudo que for relevante para repetir,
  auditar ou corrigir a execução.
- Limpar `.workspace/` e `.secrets/` ao final, salvo pedido explícito
  para manter algum artefato temporário.

## Arquivos de Agentes

Os arquivos `AGENTS.md`, `CLAUDE.md` e `GEMINI.md` mantêm Codex,
Claude Code e Gemini alinhados sobre as mesmas regras do repositório.
Quando uma convenção operacional mudar, atualizar os três arquivos ou
mantê-los apontando para este README como fonte principal.
