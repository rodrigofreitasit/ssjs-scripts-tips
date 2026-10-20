# Data 360 Config Snapshot: spec para o Kiro

Spec para recriar no [Kiro](https://kiro.dev) uma ferramenta local (CLI) que gera um snapshot somente leitura da configuração de uma org Salesforce Data 360 (Data Cloud). O snapshot cobre data model, data streams, mapeamentos, segmentos, ativações, calculated insights, identity resolution e data graphs, e é salvo em JSON normalizado.

Baseado no projeto open source [config-snapshot-data360](https://github.com/szymonlewandowski/config-snapshot-data360) (EUPL-1.2), de Szymon Lewandowski.

## Arquivos

| Arquivo | Conteúdo |
|---|---|
| `.kiro/specs/d360-config-snapshot/requirements.md` | 20 requisitos com critérios de aceite em EARS (autenticação via `sf` CLI ou token `sid`, segurança, paginação, módulos, bundle, redact, testes) |
| `.kiro/steering/data360-api.md` | Comportamento real da API `/ssot/*` e peculiaridades não documentadas pela Salesforce (verificado em API v67.0) |

## Como usar

1. Copie a pasta `.kiro/` para a raiz de um projeto novo.
2. Abra o projeto no Kiro e peça para gerar o `design.md` e o `tasks.md` da spec `d360-config-snapshot`.
