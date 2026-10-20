# Requirements Document

## Introduction

Ferramenta local, em linha de comando, que gera um **snapshot somente leitura da configuração de uma org Salesforce Data 360 (Data Cloud)**. O snapshot cobre modelo de dados, data streams, mapeamentos, segmentos, ativações, calculated insights, identity resolution e data graphs, e é salvo como um `.zip` de JSON normalizado e determinístico.

Usos previstos:
- documentar a configuração de uma org;
- revisar a configuração antes ou depois de um deploy;
- comparar o estado da org ao longo do tempo, versionando os snapshots em git.

A ferramenta roda na máquina do usuário e só se comunica com a própria org Salesforce. Não tem servidor, banco de dados nem telemetria.

Referência funcional: extensão open source "Config Snapshot for Salesforce Data 360" (github.com/szymonlewandowski/config-snapshot-data360, schema 0.5, verificada na API v67.0). O arquivo de steering `data360-api.md` documenta os endpoints e o comportamento real da API. **Os requisitos abaixo dependem dele.**

### Glossário

- **Org**: instância Salesforce com Data 360 habilitado.
- **My Domain host**: `https://<dominio>.my.salesforce.com` (ou `<dominio>.sandbox.my.salesforce.com`, `<dominio>.develop.my.salesforce.com`).
- **Data space**: partição lógica do Data 360. Um snapshot cobre exatamente um data space.
- **DMO / DLO**: Data Model Object (`__dlm`) / Data Lake Object (`__dll`).
- **CI**: Calculated Insight (`__cio`).
- **Módulo**: cada uma das 8 áreas exportadas: `model`, `ingestion`, `mappings`, `segments`, `activations`, `calculatedInsights`, `identityResolution`, `dataGraphs`.
- **Bundle**: o `.zip` gerado.
- **Cliente HTTP**: componente único pelo qual passam todas as chamadas à org.

### Restrições gerais

- Node.js 20+ com TypeScript; a CLI distribuída é JavaScript compilado.
- Sem dependências de runtime, se possível (usar `fetch`, `zlib` e `node:util.parseArgs` nativos). Dependências só de desenvolvimento são aceitáveis.
- O núcleo de exportação (cliente HTTP, módulos, normalização, bundle) não pode depender de APIs de Node além de `fetch` injetado. Assim ele pode ser reaproveitado em uma extensão de navegador no futuro.

## Requirements

### Requirement 1: Autenticação via Salesforce CLI

**User Story:** Como arquiteto de CRM, quero usar a sessão que já tenho no Salesforce CLI, para não manipular tokens manualmente.

#### Acceptance Criteria

1. WHEN o usuário informar `--sf-org <alias>` THEN o sistema SHALL executar `sf org display --target-org <alias> --json` e usar `result.accessToken` e `result.instanceUrl`.
2. IF o comando `sf` não existir, retornar código diferente de zero ou não trouxer `accessToken`/`instanceUrl` THEN o sistema SHALL encerrar com código 1 e uma mensagem que inclua o comando de login sugerido (`sf org login web --alias <alias> --instance-url <url>`).
3. The sistema SHALL ler a saída JSON do `sf` mesmo quando o processo terminar com erro, para mostrar a mensagem original do CLI.

### Requirement 2: Autenticação por token de sessão (equivalente ao cookie `sid`)

**User Story:** Como usuário sem o Salesforce CLI, quero colar o valor do cookie `sid` da minha sessão no navegador, para gerar o snapshot do mesmo jeito que a extensão faz.

#### Acceptance Criteria

1. WHEN as variáveis de ambiente `SF_INSTANCE_URL` e `SF_ACCESS_TOKEN` (ou `SF_SID`) estiverem definidas, ou as flags `--url` e `--token` forem usadas, THEN o sistema SHALL usá-las como credencial. As flags têm precedência sobre as variáveis de ambiente.
2. WHEN a URL informada for de Lightning (`*.lightning.force.com`), Setup (`*.my.salesforce-setup.com`) ou My Domain (`*.my.salesforce.com`) THEN o sistema SHALL convertê-la para o My Domain host correspondente, preservando os segmentos `sandbox` e `develop`. Exemplos:
   - `acme.lightning.force.com` → `acme.my.salesforce.com`
   - `acme--uat.sandbox.lightning.force.com` → `acme--uat.sandbox.my.salesforce.com`
   - `x-dev-ed.develop.lightning.force.com` → `x-dev-ed.develop.my.salesforce.com`
   - `acme.my.salesforce-setup.com` → `acme.my.salesforce.com`
3. IF o prefixo do host contiver caracteres fora de `[a-z0-9-.]` THEN o sistema SHALL rejeitar a URL.
4. IF nenhuma credencial for fornecida THEN o sistema SHALL encerrar com código 1 e indicar as duas formas de autenticação.
5. The documentação SHALL explicar que o cookie `sid` precisa ser o do domínio `*.my.salesforce.com`, porque o do `*.lightning.force.com` retorna 401 `INVALID_SESSION_ID`.

### Requirement 3: Segurança do token e do tráfego

**User Story:** Como responsável pela segurança, quero garantir que a ferramenta não altere nada na org e nunca exponha o token, para poder aprová-la num ambiente regulado.

#### Acceptance Criteria

1. The Cliente HTTP SHALL fazer apenas requisições `GET`. Qualquer outro método SHALL ser impossível pela interface do cliente.
2. The Cliente HTTP SHALL enviar o token somente no cabeçalho `Authorization: Bearer <token>`, junto com `Accept: application/json`.
3. The Cliente HTTP SHALL não seguir redirects (`redirect: 'manual'`). WHEN a resposta for 3xx ou `opaqueredirect` THEN o resultado SHALL ser o erro `REDIRECT`.
4. IF uma URL a ser chamada (inclusive uma `nextPageUrl` absoluta retornada pela API) tiver origem diferente do instance URL THEN o Cliente HTTP SHALL recusar a chamada sem enviar o token.
5. The sistema SHALL nunca gravar o token em disco, em log ou em mensagens de erro.
6. The sistema SHALL não enviar dados a nenhum destino além da org.

### Requirement 4: Resiliência do cliente HTTP

**User Story:** Como usuário, quero que falhas transitórias da API não derrubem o snapshot inteiro.

#### Acceptance Criteria

1. WHEN a resposta for 429, 502, 503 ou 504, houver erro de rede ou estourar o timeout THEN o Cliente HTTP SHALL tentar de novo até 2 vezes, com backoff exponencial (500 ms × 2^tentativa).
2. WHEN houver cabeçalho `Retry-After` numérico THEN o Cliente HTTP SHALL esperar esse valor, limitado a 30 s.
3. The Cliente HTTP SHALL aplicar timeout de 90 s por tentativa, cobrindo também a leitura do corpo.
4. IF uma resposta 2xx não for JSON válido (página de manutenção, login ou proxy) THEN o resultado SHALL ser o erro `NON_JSON_RESPONSE`, e não um dado vazio.
5. The Cliente HTTP SHALL devolver `{ ok, status, body, errorCode, message, path, durationMs }`. Em erro, `errorCode` vem de `body[0].errorCode` ou `body.error`; na falta deles, `HTTP_<status>`.
6. WHEN o usuário pressionar Ctrl+C THEN o sistema SHALL cancelar as requisições em andamento (AbortSignal), não gravar arquivo e encerrar com código 130.

### Requirement 5: Conexão e validação da sessão

**User Story:** Como usuário, quero saber logo no início se a sessão é válida e qual org e usuário estão sendo usados.

#### Acceptance Criteria

1. WHEN o sistema iniciar THEN ele SHALL chamar `GET /services/data/` e escolher a maior versão entre a mínima (61.0) e a testada (67.0, configurável numa constante).
2. IF não houver versão nesse intervalo THEN o sistema SHALL encerrar informando o intervalo suportado.
3. The sistema SHALL validar o token chamando `/limits`, e não apenas `/services/data/`, que responde sem autenticação. Em paralelo, SHALL chamar `/services/oauth2/userinfo` e `/ssot/data-spaces?limit=200`.
4. WHEN a conexão tiver sucesso THEN o sistema SHALL exibir org id, usuário, versão da API, host, origem da credencial, chamadas diárias restantes/máximas (`DailyApiRequests`) e data spaces disponíveis.
5. IF `/ssot/data-spaces` falhar THEN o sistema SHALL informar que o Data 360 não está disponível. IF a lista vier vazia THEN o sistema SHALL assumir `default`.
6. The sistema SHALL traduzir os erros de sessão em mensagens claras: `INVALID_SESSION_ID` (sessão expirada), `API_DISABLED_FOR_ORG`/`API_CURRENTLY_DISABLED` (falta a permissão API Enabled), 403 (sem permissão no Data 360), 404 (Data 360 não habilitado), `REDIRECT`, `TIMEOUT`, `NETWORK_ERROR` e `NON_JSON_RESPONSE`.
7. WHEN a flag `--list-spaces` for usada THEN o sistema SHALL apenas conectar, exibir as informações e encerrar.

### Requirement 6: Seleção de escopo e estimativa de chamadas

**User Story:** Como usuário, quero escolher o data space e os módulos e ver quantas chamadas serão feitas antes de exportar, para não gastar o limite diário de API sem querer.

#### Acceptance Criteria

1. The sistema SHALL aceitar `--data-space <nome>` (padrão `default`). IF o data space não existir THEN o sistema SHALL encerrar listando os disponíveis.
2. The sistema SHALL aceitar `--modules a,b,c` (padrão: todos). IF um módulo for desconhecido THEN o sistema SHALL encerrar listando os válidos.
3. Before exportar, o sistema SHALL fazer um inventário com uma única chamada a `/ssot/metadata-entities` e mostrar a estimativa de requisições por módulo e total:
   - model: 1
   - ingestion: 2 + 1 por DLO (aproximado)
   - mappings: 1 por DMO no escopo
   - segments: 1 por 100 segmentos
   - activations: 2 + 1 por ativação
   - calculatedInsights: 4
   - identityResolution: 1
   - dataGraphs: 1 + 1 por graph
4. WHEN a estimativa for exibida em terminal interativo THEN o sistema SHALL pedir confirmação. `--yes`, ou stdin sem TTY, pula a confirmação.
5. IF o inventário falhar THEN o sistema SHALL avisar e seguir sem estimativa.

### Requirement 7: Orquestração da coleta

**User Story:** Como usuário, quero que a exportação seja rápida sem sobrecarregar a org, e que a falha de um módulo não perca os outros.

#### Acceptance Criteria

1. The sistema SHALL limitar as requisições simultâneas, somando todos os módulos, a `--concurrency` (padrão 4).
2. The sistema SHALL executar `model` primeiro, porque os outros módulos reaproveitam a resposta de `/ssot/metadata`, e depois os demais em paralelo.
3. WHEN um módulo falhar THEN o sistema SHALL registrar o erro em `manifest.modules.<módulo>.error` e continuar os demais.
4. The sistema SHALL registrar por módulo `fetchedAt`, `calls` e `durationMs`, e mostrar o progresso no stderr (`[n/total] módulo — X calls, Y s`).
5. IF nenhum módulo tiver sucesso THEN o sistema SHALL não gravar nenhum arquivo e encerrar com código 1.
6. IF algum módulo falhar e outros tiverem sucesso THEN o sistema SHALL gravar o bundle e encerrar com código 2.

### Requirement 8: Paginação correta

**User Story:** Como usuário, quero ter certeza de que nenhum item foi omitido por paginação inconsistente da API.

#### Acceptance Criteria

1. The sistema SHALL paginar cada endpoint conforme a seção "Paginação" do steering. Em especial:
   - seguir `nextPageUrl` em `/ssot/data-lake-objects` e `/ssot/data-streams`, sem usar "página curta = fim";
   - usar `batchSize`/`offset` em segments, activations e activation-targets;
   - usar `pageToken` em calculated insights.
2. The paginador SHALL ter limite máximo de páginas (500), detectar páginas repetidas e remover itens duplicados.
3. WHEN houver truncamento, repetição ou duplicata THEN o módulo SHALL registrar o warning `PAGING_TRUNCATED`, `PAGING_REPEATED` ou `PAGING_DUPLICATES`.

### Requirement 9: Módulo `model` (modelo de dados)

**User Story:** Como arquiteto, quero a lista de DMOs realmente usados, com campos, chaves e relacionamentos ativos.

#### Acceptance Criteria

1. The módulo SHALL usar `/ssot/metadata?entityType=DataModelObject` (com `dataspace` quando não for `default`) e não o catálogo `/ssot/data-model-objects`, que é lento e traz DMOs não usados.
2. The módulo SHALL excluir os DMOs das categorias `Segment_Membership`, `Activation_Audience` e `Vector_Embedding`, e os DMOs de sistema de data graphs, identificados pelos padrões de nome do steering.
3. For cada DMO, o módulo SHALL gerar `name`, `label`, `category`, `standard` (prefixo `ssot__`), `derived` (`"identityResolution"` para `Unified*` e `*IdentityLink__dlm`; senão `null`), `primaryKey[]` e `fields[]` com `name`, `label`, `type`, `businessType` e `system`.
4. The módulo SHALL gerar `relationships[]` como `{ from: {dmo, field}, to: {dmo, field}, cardinality }`, sem duplicar a relação que aparece nas duas pontas (chave `from|field|to|field`).

### Requirement 10: Módulo `ingestion` (data streams e DLOs)

#### Acceptance Criteria

1. The módulo SHALL listar `/ssot/data-streams` e `/ssot/data-lake-objects`, e buscar o detalhe de cada stream para obter os mapeamentos de campo, que não vêm na lista nem com `includeMappings=true`.
2. Como esses endpoints não aceitam `dataspace`, o módulo SHALL filtrar por `dataSpaceInfo`, mantendo itens sem essa informação.
3. For cada stream, o módulo SHALL gerar `name`, `label`, `connector {type, name, sourceObject}`, `dlo`, `status`, `refreshMode`, `lastRunStatus`, `lastRefreshDate`, `system` (true para o conector `AIPlatform`) e `fieldMappings[] {sourceField, dloField}`.
4. WHEN o mapeamento nomear o campo do DLO sem o sufixo `__c` THEN o módulo SHALL procurar primeiro o nome exato, depois `+__c`, e gravar o nome completo.
5. For cada DLO, o módulo SHALL gerar `name`, `label`, `category`, `primaryKey[]` e `fields[] {name, label, type, isPrimaryKey, system}`.

### Requirement 11: Módulo `mappings` (DLO → DMO)

#### Acceptance Criteria

1. The módulo SHALL chamar `/ssot/data-model-object-mappings?dmoDeveloperName=<dmo>` para cada DMO no escopo do `model`, exceto DMOs excluídos e derivados.
2. WHEN a resposta for 404 `NOT_FOUND` THEN o módulo SHALL tratar como "DMO sem mapeamentos", e não como erro.
3. The módulo SHALL gerar `mappings[] {name, dlo, dmo, status, fieldMappings[] {dloField, dmoField, system}}`.

### Requirement 12: Módulo `segments`

#### Acceptance Criteria

1. The módulo SHALL listar `/ssot/segments` com `batchSize=100` e `offset`, usando o `includeCriteria` da própria lista, sem chamada de detalhe.
2. The módulo SHALL decodificar as entidades HTML de `includeCriteria` (`&quot;`, `&amp;`, `&#NN;`, etc.) antes de fazer o parse do JSON.
3. The módulo SHALL converter o critério numa árvore com os nós `and`/`or`, `compare`, `aggregate`, `calculatedInsight`, `waterfall`, `segment` e `raw` (para tipos desconhecidos, mantidos sem alteração), conforme o steering.
4. IF o critério vier vazio THEN o módulo SHALL marcar `criteriaAvailable: false` e registrar o warning `SEGMENT_NO_CRITERIA`, sem inventar critério.
5. For cada segmento, o módulo SHALL gerar `name`, `label`, `type`, `segmentOn`, `status`, `parameters[]` (segmentos dinâmicos usam `parameterizedValues`), `criteria`, `fieldRefs[] {object, field}`, `segmentRefs[]` e `unknownNodeTypes[]`.

### Requirement 13: Módulo `activations`

#### Acceptance Criteria

1. The módulo SHALL listar `/ssot/activations` e `/ssot/activation-targets` com `batchSize`/`offset`, filtrar por `dataSpaceName` e buscar o detalhe de cada ativação **por `id`**, nunca por nome.
2. The módulo SHALL decodificar `contactPointPath` (JSON com entidades HTML) e ler `objectName`/`fieldName` nos caminhos e filtros.
3. The módulo SHALL gerar `activations[]` com `name`, `label`, `type` (SEGMENT/DMO/API_TRIGGERED), `status`, `subject`, `segment`, `target`, `targetPlatform`, `refreshType`, `processingType`, `lastPublishStatus`, `lastPublishDate`, `audienceDmos {history, latest}`, `attributes[]`, `contactPoints[]`, `filters {direct[], related[]}`, `staticData[]` e `fieldRefs[]`.
4. The módulo SHALL gerar `activationTargets[] {name, id, platformName, platformType, status, audienceDmos}`.

### Requirement 14: Módulo `calculatedInsights`

#### Acceptance Criteria

1. The módulo SHALL chamar `/ssot/calculated-insights` uma vez para cada `definitionType` (`CALCULATED_METRIC`, `STREAMING_METRIC`, `EXTERNAL_METRIC`), porque sem o parâmetro a API retorna 0 itens. Também SHALL chamar `/ssot/metadata?entityType=CalculatedInsight` para obter tipos e objetos de origem.
2. The módulo SHALL gerar `name`, `label`, `definitionType`, `status`, `creationType`, `description`, `expression` (SQL), `dimensions[]`, `measures[]`, `schedule`, `lastRunStatus`, `lastRunDate`, `sourceObjects[]` e `fieldRefs[]`.
3. The módulo SHALL extrair `fieldRefs` do SQL pelo padrão `objeto__dlm.campo`. A documentação SHALL registrar a limitação: aliases de tabela e campos sem prefixo não são capturados.

### Requirement 15: Módulo `identityResolution`

#### Acceptance Criteria

1. The módulo SHALL usar uma única chamada a `/ssot/identity-resolutions`, filtrada por `dataSpaceName`.
2. Como a API não traz developer name, o módulo SHALL usar o `id` do registro como `name`.
3. The módulo SHALL gerar `rulesets[]` com `label`, `description`, `object`, `configurationType`, `status`, `runsAutomatically`, `primaryDmo`, `matchRules[]`, `reconciliationRules[]`, `stats {sourceProfiles, matchedSourceProfiles, totalUnifiedProfiles, consolidationRate}` e `lastJob`.

### Requirement 16: Módulo `dataGraphs`

#### Acceptance Criteria

1. The módulo SHALL chamar `/ssot/data-graphs/metadata` uma vez e `/ssot/data-graphs/{name}` para cada graph.
2. The módulo SHALL gerar `status` a partir do detalhe e `metadataStatus` a partir do metadata, porque os dois podem divergir.
3. The módulo SHALL gerar `name`, `label`, `description`, `primaryObject`, `primaryObjectType`, `refresh {full, incremental}`, `systemDmos {ids, values, fragments[]}` e `nodes[]` em ordem depth-first, com `object`, `type`, `parent`, `depth`, `join[]`, `fields[]`, `filter`, `recency[]` (pode ter condição de tempo e de quantidade ao mesmo tempo) e `fragmentDmo`.

### Requirement 17: Normalização e verificação entre módulos

**User Story:** Como usuário que versiona snapshots em git, quero que dois snapshots da mesma org gerem diffs limpos e que referências quebradas sejam apontadas.

#### Acceptance Criteria

1. The sistema SHALL ordenar toda coleção e toda lista de campos por `name`, e escrever os módulos sempre na mesma ordem, independentemente da ordem de conclusão.
2. The sistema SHALL guardar as referências entre itens pelo nome de API (`name`). Os labels servem só para exibição.
3. The sistema SHALL marcar com `system: true` os campos técnicos (`KQ_*`, `cdp_sys_*`, `DataSource__c`, `DataSourceObject__c`, `ssot__DataSourceId__c`, `ssot__DataSourceObjectId__c`) e os streams de sistema.
4. WHEN os módulos envolvidos estiverem no snapshot THEN o sistema SHALL verificar as referências entre eles e registrar em `manifest.linkWarnings[]` os códigos `MISSING_DLO`, `MISSING_DMO`, `MISSING_FIELD`, `MISSING_SEGMENT` etc. A falta de um módulo nunca é erro.
5. The normalização SHALL funcionar offline a partir das respostas brutas, para permitir reprocessar um bundle que tenha `raw/`.

### Requirement 18: Bundle de saída

#### Acceptance Criteria

1. The sistema SHALL gravar `<out>/d360-<myDomain>-<dataSpace>-<YYYYMMDD-HHmm>.zip`, com `--out` padrão `./snapshots`. `<myDomain>` é o primeiro rótulo do host, com até 30 caracteres; na falta dele, usa-se o org id.
2. The zip SHALL conter `manifest.json`, `snapshot/<módulo>.json` (JSON indentado, no formato `{ module, schemaVersion, ...coleções, warnings[] }`) e `README.txt`. Com `--raw`, SHALL conter também `raw/<módulo>.json` (JSON compacto).
3. The `manifest.json` SHALL conter `bundle: "data360-export"`, `bundleVersion`, `schemaVersion`, `createdAt`, `source {orgId, apiVersion, dataSpace}`, `redacted`, `modules{}`, `counts{}` (tamanho de cada coleção por módulo) e `linkWarnings[]`.
4. The zip SHALL usar DEFLATE.
5. WHEN `--dir` for usado THEN o sistema SHALL gravar também os mesmos arquivos descompactados em `<out>/<nome-sem-.zip>/`.
6. WHEN o sistema terminar com sucesso THEN ele SHALL imprimir no stdout apenas o caminho do zip, com todo o resto no stderr, para permitir uso em scripts.

### Requirement 19: Mascaramento (redact)

**User Story:** Como usuário, quero compartilhar um snapshot fora do time sem expor valores de negócio ou dados pessoais presentes em filtros.

#### Acceptance Criteria

1. WHEN `--redact` for usado THEN o sistema SHALL substituir por `<redacted>`:
   - os valores dos nós `compare` em segmentos e filtros de ativação;
   - os nós `raw` inteiros;
   - o `value` de `staticData`;
   - os literais de string SQL (`'...'`, incluindo escape `''`) em `expression` e `formula`;
   - o `filter` dos nós de data graph.
2. The mascaramento SHALL preservar objetos, campos, operadores e joins.
3. WHEN `--redact` for usado THEN o sistema SHALL ignorar `--raw`, porque `raw/` contém os originais, e SHALL registrar `redacted: true` no manifest.

### Requirement 20: Qualidade e testes

#### Acceptance Criteria

1. The projeto SHALL incluir uma org sintética (fake `fetch`) que reproduza as respostas e as peculiaridades do steering, e permitir rodar a CLI contra ela com `--fake`.
2. The testes SHALL cobrir pelo menos:
   - mapeamento de URLs para o My Domain host;
   - recusa de outra origem e de redirect;
   - retry e `Retry-After`;
   - `NON_JSON_RESPONSE`;
   - paginação de cada estilo, incluindo `nextPageUrl` em páginas curtas, e os warnings de paginação;
   - exclusão de DMOs de sistema;
   - restauração do sufixo `__c`;
   - parse de critérios com entidades HTML;
   - 404 em mappings tratado como vazio;
   - ordenação determinística (duas execuções produzem JSON idêntico, exceto datas);
   - redact;
   - `linkWarnings`.
3. The testes SHALL rodar com `node --test` e sem acesso à rede.

## Fora de escopo (versão 1)

- Qualquer escrita na org (deploy, restore, criação de metadados).
- Mais de um data space por execução (o usuário roda uma vez por data space).
- Contagem de registros ou população de segmentos.
- Mapeamentos por fórmula (transform), que não estão disponíveis nesta API.
- Diff entre dois snapshots. É candidato a uma próxima spec: o formato determinístico já permite `git diff`.
- Autenticação OAuth própria (Connected App ou External Client App com client credentials). É candidata a uma próxima spec, para execução agendada sem usuário.
