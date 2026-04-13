# DAS — Plataforma de Geofencing Inteligente

**Documento de Arquitetura de Software**

|Campo   |Valor     |
|--------|----------|
|Versão  |1.1.0     |
|Data    |Abril 2026|
|Status  |Draft     |
|Cloud   |AWS       |
|Usuários|1M+       |


> Arquitetura serverless event-driven para detecção de presença geográfica, motor de elegibilidade e engajamento omnichannel em tempo real.

-----

## 01 — Visão Geral

A plataforma permite detectar quando usuários do app mobile entram ou saem de áreas geográficas predefinidas (geofences) e, com base em regras de elegibilidade configuráveis, acionar comunicações multi-canal via Salesforce Marketing Cloud (Push, WhatsApp, E-mail, SMS) e Firebase Cloud Messaging, com suporte a estratégias de orquestração entre canais (fan-out simultâneo ou cascata com delay).

### Princípios Arquiteturais

- **Detecção no Device**: geofences registradas localmente no Android (GeofencingClient) e iOS (CLLocationManager). Processamento on-device sem polling no servidor — zero custo de infra para a detecção.
- **Processamento Event-Driven**: eventos publicados via API Gateway → Lambda → EventBridge. Arquitetura desacoplada onde cada consumer (notificação, analytics, CRM) é independente.
- **Motor de Elegibilidade**: regras configuráveis por geofence — segmentos de usuário, cooldown, limite mensal, horário, campanha ativa. Toda lógica no backend, sem redeploy mobile.
- **Engajamento Omnichannel**: integração com Salesforce Marketing Cloud (Journey Builder) e Firebase (Push). Fan-out via EventBridge para múltiplos canais — Push, WhatsApp, SMS, E-mail — com orquestração configurável (simultâneo ou cascata com delay).

### Visão Macro

```
┌──────────────────┐       ┌──────────────────────────────────────────────────────────┐       ┌──────────────────────┐
│   MOBILE LAYER   │       │                AWS BACKEND (Serverless)                  │       │  CANAIS DE COMUNIC.  │
│                  │       │                                                          │       │                      │
│  Android         │       │  API Gateway ──▶ Lambda Processor                        │       │  ┌─ Firebase FCM     │
│  GeofencingClient│──────▶│       │              │                                   │       │  │  (Push imediato)   │
│                  │ HTTPS │       ▼              ▼                                   │       │  │                    │
│  iOS             │ POST  │  Lambda Eligibility  DynamoDB (4 tabelas)                │       │  ├─ Marketing Cloud  │
│  CLLocationMgr   │       │       │                                                  │──────▶│  │  Journey Builder  │
│                  │       │       ▼                                                  │       │  │   ├─ WhatsApp     │
│  Geofence local  │       │  EventBridge ─┬──▶ SQS Push ──▶ Lambda Firebase Conn.    │       │  │   ├─ Email        │
│                  │       │               ├──▶ SQS WA ────▶ Lambda MC WA Conn.       │       │  │   └─ SMS          │
│                  │       │               ├──▶ SQS Email ─▶ Lambda MC Email Conn.    │       │  │                    │
│                  │       │               └──▶ SQS Analyt.▶ Lambda S3/Athena         │       │  └─ Analytics        │
└──────────────────┘       └──────────────────────────────────────────────────────────┘       └──────────────────────┘
```

-----

## 02 — Componentes

|Componente                |Tipo    |Tecnologia                   |Responsabilidade                                                                                     |
|--------------------------|--------|-----------------------------|-----------------------------------------------------------------------------------------------------|
|**Geofence SDK — Android**|Mobile  |Kotlin + Google Play Services|Registra geofences localmente, detecta ENTER/EXIT, envia evento ao backend                           |
|**Geofence SDK — iOS**    |Mobile  |Swift + Core Location        |CLCircularRegion monitoring, detecção em background, envio do evento                                 |
|**API Gateway**           |Infra   |AWS API Gateway (REST)       |Ingress HTTPS, rate limiting, autenticação via API Key + JWT                                         |
|**Geofence Processor**    |Lambda  |Node.js / Python             |Valida payload, persiste evento, invoca Eligibility Engine                                           |
|**Eligibility Engine**    |Lambda  |Node.js / Python             |Avalia regras de elegibilidade, decide se usuário recebe comunicação e quais canais acionar          |
|**EventBridge**           |Evento  |AWS EventBridge              |Roteamento event-driven com fan-out para múltiplas filas por canal (Push, WhatsApp, Email, Analytics)|
|**SQS Push Queue**        |Fila    |AWS SQS                      |Buffer para notificações push via Firebase                                                           |
|**SQS WhatsApp Queue**    |Fila    |AWS SQS                      |Buffer para mensagens WhatsApp via Marketing Cloud                                                   |
|**SQS Email Queue**       |Fila    |AWS SQS                      |Buffer para e-mails via Marketing Cloud                                                              |
|**SQS Analytics Queue**   |Fila    |AWS SQS                      |Buffer para ingestão de eventos em S3/Athena                                                         |
|**Firebase Connector**    |Lambda  |Node.js / Python             |Dispara push via Firebase Admin SDK para notificação imediata                                        |
|**MC WhatsApp Connector** |Lambda  |Node.js / Python             |Chama Journey Builder API do Marketing Cloud para acionar canal WhatsApp                             |
|**MC Email Connector**    |Lambda  |Node.js / Python             |Chama Journey Builder API do Marketing Cloud para acionar canal Email                                |
|**MC Journey Connector**  |Lambda  |Node.js / Python             |Chama Journey Builder API para fluxos de cascata orquestrados (push → delay → WhatsApp)              |
|**Analytics Connector**   |Lambda  |Node.js / Python             |Grava eventos em S3 (Parquet) para análise via Athena/QuickSight                                     |
|**Admin API**             |Infra   |API Gateway + Lambda         |CRUD de geofences, regras de elegibilidade e configuração de canais. Endpoint privado (admin)        |
|**Admin Web App**         |Frontend|React + Mapbox/Google Maps   |Interface para cadastro de geofences no mapa, configuração de regras e seleção de canais/estratégia  |
|**DynamoDB**              |Storage |AWS DynamoDB                 |4 tabelas: Geofences, EligibilityRules, GeofenceEvents, UserGeofenceHistory                          |

-----

## 03 — Modelo de Dados

Modelagem DynamoDB single-table por domínio, otimizada para os padrões de acesso da plataforma.

### 3.1 Tabela: Geofences

Armazena as definições de cada cerca geográfica cadastrada no admin.

```json
// Partition Key: PK | Sort Key: SK
{
  "PK": "GEOFENCE#geo_001",
  "SK": "METADATA",
  "name": "Shopping Ibirapuera",
  "latitude": -23.6115,
  "longitude": -46.6590,
  "radius_meters": 200,
  "category": "loja_propria",
  "status": "active",
  "created_at": "2026-04-12T10:00:00Z",
  "updated_at": "2026-04-12T10:00:00Z"
}
```

### 3.2 Tabela: EligibilityRules

Regras de elegibilidade vinculadas a cada geofence. Suporta múltiplas regras por geofence.

```json
{
  "PK": "GEOFENCE#geo_001",
  "SK": "RULE#rule_001",
  "rule_name": "Campanha Inverno Premium",
  "conditions": {
    "segments": ["premium", "gold"],
    "min_app_version": "4.2.0",
    "opt_in_push": true,
    "cooldown_hours": 24,
    "max_triggers_month": 3,
    "active_campaign": "camp_inverno_2026",
    "day_of_week": ["MON", "TUE", "WED", "THU", "FRI"],
    "hour_range": [9, 21]
  },
  "notification_config": {
    "channels": ["push", "whatsapp", "email"],
    "strategy": "cascade",
    "cascade_delay_minutes": 10,
    "journey_key": "GEOFENCE_ENTRY_WINTER",
    "fallback_channel": "whatsapp",
    "channel_configs": {
      "push": {
        "connector": "firebase",
        "priority": 1
      },
      "whatsapp": {
        "connector": "marketing_cloud",
        "journey_key": "GEOFENCE_WA_WINTER",
        "template_name": "geofence_welcome_winter",
        "priority": 2
      },
      "email": {
        "connector": "marketing_cloud",
        "journey_key": "GEOFENCE_EMAIL_WINTER",
        "priority": 3
      }
    }
  },
  "priority": 1,
  "status": "active"
}
```

### 3.3 Tabela: GeofenceEvents

Log de todos os eventos detectados. Usado para analytics, auditoria e cálculo de cooldown.

```json
{
  "PK": "USER#usr_abc123",
  "SK": "EVENT#2026-04-12T14:30:00Z",
  "geofence_id": "geo_001",
  "event_type": "ENTER",
  "latitude": -23.6118,
  "longitude": -46.6587,
  "eligible": true,
  "rule_matched": "rule_001",
  "notification_sent": true,
  "channels_triggered": ["push", "whatsapp"],
  "channel_results": {
    "push": { "status": "delivered", "sent_at": "2026-04-12T14:30:01Z" },
    "whatsapp": { "status": "delivered", "sent_at": "2026-04-12T14:40:01Z" }
  },
  "strategy_used": "cascade",
  "blocked_reason": null,
  "device_os": "android",
  "app_version": "4.3.1",
  "ttl": 1744502400  // TTL 90 dias
}
```

### 3.4 Tabela: UserGeofenceHistory

Controle de cooldown e throttle por par usuário+geofence. Atualizada atomicamente.

```json
{
  "PK": "USER#usr_abc123",
  "SK": "GEOFENCE#geo_001",
  "last_triggered_at": "2026-04-12T14:30:00Z",
  "trigger_count_month": 2,
  "trigger_count_reset_at": "2026-05-01T00:00:00Z",
  "total_visits": 15,
  "first_visit_at": "2026-01-15T09:12:00Z"
}
```

-----

## 04 — Motor de Elegibilidade

Pipeline de avaliação sequencial que decide se um evento de geofence deve gerar comunicação. Toda a lógica reside no backend — mudanças de regra não exigem release mobile.

### Pipeline de Avaliação

**Passo 1 — Recebe evento ENTER**
Lambda Processor valida o payload (user_id, geofence_id, event_type, timestamp, lat/lng). Rejeita eventos duplicados via idempotency key (user_id + geofence_id + timestamp truncado em 5 min).

**Passo 2 — Busca geofence + regras**
Query no DynamoDB com PK = `GEOFENCE#{id}`, trazendo metadata e todas as rules. Valida se geofence está ativa. Ordena regras por prioridade.

**Passo 3 — Busca perfil do usuário**
Recupera dados do usuário: segmento, opt-in, versão do app. Pode vir do DynamoDB (cache) ou chamada ao CRM. Para 1M+ usuários, recomenda-se cache local no DynamoDB com TTL de 24h.

**Passo 4 — Avalia condições (fail-fast)**
Cada condição é avaliada em sequência — rejeita na primeira falha:

1. **Segmento** — usuário pertence a um dos segmentos da regra?
1. **Opt-in** — usuário aceitou receber push/comunicações?
1. **Versão do App** — versão >= min_app_version?
1. **Horário** — momento atual dentro de hour_range e day_of_week?
1. **Campanha Ativa** — campanha associada está vigente?
1. **Cooldown** — última notificação nessa geofence foi há mais de X horas?
1. **Throttle Mensal** — trigger_count_month < max_triggers_month?

**Passo 5 — Decisão e Roteamento Multi-Canal**

- **Elegível:** publica evento qualificado no EventBridge com a lista de canais e estratégia definida na regra. EventBridge faz fan-out para as filas SQS de cada canal. Atualiza UserGeofenceHistory.
- **Não elegível:** loga evento com `blocked_reason` (ex: `cooldown_active`, `segment_mismatch`) para analytics.

**Passo 6 — Orquestração entre canais**

A estratégia de entrega é definida na `notification_config` de cada regra:

- **Fan-out simultâneo (`strategy: "fanout"`):** EventBridge dispara todas as filas ao mesmo tempo. Cada canal (Push, WhatsApp, Email) recebe o evento independentemente. Usado quando todos os canais devem ser acionados juntos.
- **Cascata com delay (`strategy: "cascade"`):** apenas o canal de prioridade 1 (ex: Push) é acionado imediatamente. O Marketing Cloud Journey Builder orquestra o restante — aguarda `cascade_delay_minutes`, verifica se o usuário interagiu (abriu push), e só então aciona o canal de prioridade 2 (ex: WhatsApp). Isso evita bombardear o usuário e usa canais mais caros (WhatsApp/SMS) apenas como fallback.
- **Journey orquestrado (`strategy: "journey"`):** toda a lógica de canais é delegada a um único Journey no Marketing Cloud. O backend envia um evento de entrada no Journey e o MC decide internamente quais canais usar, com base em preferências do contato, horário, e regras de frequência do próprio MC.

> **Regras compostas:** cada geofence pode ter múltiplas regras com prioridades diferentes. Se a regra de prioridade 1 rejeita, a engine tenta a regra de prioridade 2 — permitindo cenários como “se não é premium, tenta a campanha genérica”.

-----

## 05 — Fluxos Principais

### 5.1 Fluxo A — Evento de Geofence com Fan-out Multi-Canal

|# |Origem              |Destino                  |Ação                                                          |Latência Alvo|
|--|--------------------|-------------------------|--------------------------------------------------------------|-------------|
|1 |Device (Android/iOS)|API Gateway              |POST `/v1/geofence-events` com payload JWT-assinado           |—            |
|2 |API Gateway         |Lambda Processor         |Valida JWT, verifica rate limit, invoca Lambda                |< 50ms       |
|3 |Lambda Processor    |DynamoDB                 |Grava evento cru na tabela GeofenceEvents                     |< 10ms       |
|4 |Lambda Processor    |Lambda Eligibility       |Invoca Engine de elegibilidade (sync)                         |< 100ms      |
|5 |Lambda Eligibility  |DynamoDB                 |Busca geofence, regras, perfil do user, histórico             |< 20ms       |
|6 |Lambda Eligibility  |EventBridge              |Publica evento qualificado com `channels` e `strategy`        |< 15ms       |
|7 |EventBridge         |SQS (múltiplas)          |Fan-out para filas por canal: Push, WhatsApp, Email, Analytics|< 10ms       |
|8a|SQS Push            |Lambda Firebase Connector|Dispara push imediato via Firebase Admin SDK                  |< 200ms      |
|8b|SQS WhatsApp        |Lambda MC WA Connector   |Chama Journey Builder API → WhatsApp Business API             |< 500ms      |
|8c|SQS Email           |Lambda MC Email Connector|Chama Journey Builder API → Email SMTP                        |< 500ms      |
|8d|SQS Analytics       |Lambda Analytics         |Grava evento em S3 (Parquet) para Athena                      |< 100ms      |
|9 |MC / Firebase       |Usuário                  |Usuário recebe push, WhatsApp e/ou email                      |1–30s        |


> **Latência end-to-end alvo:** < 3 segundos entre detecção no device e recebimento da primeira notificação. O gargalo é a API do Marketing Cloud — Firebase como canal primário para push imediato.

### 5.2 Fluxo B — Sincronização de Geofences

|#|Origem            |Destino        |Ação                                                   |
|-|------------------|---------------|-------------------------------------------------------|
|1|Admin Web App     |Admin API      |Cria/edita/remove geofence via painel com mapa         |
|2|Admin API (Lambda)|DynamoDB       |Persiste mudanças na tabela Geofences                  |
|3|Lambda            |SNS Topic      |Publica evento de atualização de geofences             |
|4|SNS               |Lambda Sync    |Gera payload atualizado (lista de geofences ativas)    |
|5|Lambda Sync       |Firebase / APNs|Envia silent push com nova lista de geofences          |
|6|Device            |Local          |App recebe push silencioso, re-registra geofences no SO|

### 5.3 Fluxo C — Cascata com Delay (Push → WhatsApp)

Fluxo detalhado quando a estratégia é `cascade` — o canal mais caro (WhatsApp) só é acionado se o push não foi efetivo.

|# |Origem               |Destino              |Ação                                                                 |Tempo  |
|--|---------------------|---------------------|---------------------------------------------------------------------|-------|
|1 |EventBridge          |SQS Push             |Aciona apenas o canal de prioridade 1 (Push)                         |t=0s   |
|2 |Lambda Firebase Conn.|Firebase FCM         |Envia push notification ao device                                    |t=1s   |
|3 |Lambda Firebase Conn.|MC Journey Builder   |Dispara evento de entrada no Journey de cascata                      |t=1s   |
|4 |MC Journey Builder   |(aguarda)            |Journey aguarda `cascade_delay_minutes` (ex: 10min)                  |t=10min|
|5 |MC Journey Builder   |(avalia)             |Verifica se usuário abriu o push (via callback ou evento de abertura)|t=10min|
|6a|Se **não** abriu     |WhatsApp Business API|MC aciona WhatsApp com template pré-aprovado                         |t=10min|
|6b|Se **abriu**         |(encerra)            |Journey finaliza sem acionar WhatsApp — usuário já engajou           |t=10min|


> **Vantagem:** reduz custo de WhatsApp em até 60-80% (apenas usuários que não engajaram com push recebem WA). Melhora experiência do usuário evitando mensagens redundantes.

-----

## 06 — Requisitos Não-Funcionais

|Requisito            |Meta     |Detalhes                                                                                                          |
|---------------------|---------|------------------------------------------------------------------------------------------------------------------|
|**Disponibilidade**  |99.9%    |Uptime mensal do pipeline de eventos. API Gateway + Lambda + DynamoDB nativamente multi-AZ.                       |
|**Latência p95**     |< 800ms  |Do POST do device até publicação no EventBridge. Exclui latência do Marketing Cloud.                              |
|**Throughput**       |10K req/s|Capacidade de burst do API Gateway. Lambda concurrency reservada: 500. Auto-scaling DynamoDB on-demand.           |
|**Retenção de Dados**|90 dias  |Eventos no DynamoDB com TTL. Dados frios exportados para S3 (Parquet) via DynamoDB Streams para análise histórica.|
|**Escalabilidade**   |1M+ users|Detecção no device (custo zero). Backend escala horizontalmente. DynamoDB on-demand sem capacity planning.        |
|**Recovery (RTO)**   |5 min    |RPO ~0 (DynamoDB multi-AZ síncrono). SQS retém mensagens por até 14 dias em caso de falha do consumer.            |

-----

## 07 — Registro de Decisões (ADR)

### ADR-001: Detecção de Geofence no Device (não no servidor)

- **Contexto:** Com 1M+ usuários, polling de localização no servidor geraria milhões de requests por minuto e alto custo de infra + bateria.
- **Decisão:** Usar APIs nativas do SO (GeofencingClient no Android, CLLocationManager no iOS). O device monitora e só notifica o backend no evento ENTER/EXIT.
- **Consequências:** Custo zero de infra para detecção. Limitação: Android suporta 100 geofences, iOS suporta 20. Com ~50 geofences, Android OK; iOS requer rotação por proximidade.
- **Alternativas descartadas:** Server-side polling (custo proibitivo), AWS Location Service (custo por device/mês alto para 1M+), third-party SDKs como Radar.io (vendor lock-in + custo).

### ADR-002: Elegibilidade avaliada no backend (não no device)

- **Contexto:** Regras de elegibilidade mudam com frequência (campanhas, segmentos, horários). Deploy mobile leva dias/semanas por conta de review das stores.
- **Decisão:** Device envia evento cru. Backend avalia todas as regras. Mudanças em regras são imediatas via painel admin sem redeploy.
- **Consequências:** Cada evento gera uma chamada ao backend (custo Lambda). Compensado pela simplicidade operacional e velocidade de iteração de negócio.
- **Alternativas descartadas:** Regras embarcadas no app (requer deploy), Remote Config do Firebase para regras (complexidade de sync, limitação de lógica).

### ADR-003: EventBridge como barramento de eventos

- **Contexto:** Eventos elegíveis precisam ser roteados para múltiplos destinos (Marketing Cloud, Firebase, analytics, futuro CRM). Acoplamento direto entre Eligibility Engine e consumers não escala.
- **Decisão:** Usar EventBridge como broker. Cada consumer é uma regra + target independente. Adicionar novos destinos sem alterar o producer.
- **Consequências:** Latência adicional ~10-15ms. Schema registry nativo. Replay de eventos em caso de falha. Custo: $1/milhão de eventos.
- **Alternativas descartadas:** SNS+SQS direto (funcional mas menos flexível para roteamento por conteúdo), Kinesis (over-engineering para ~50 geofences), Step Functions (latência alta).

### ADR-004: DynamoDB sobre Aurora/RDS

- **Contexto:** Padrões de acesso são key-value (lookup por user_id + geofence_id). Sem JOINs complexos. Necessidade de auto-scaling sem capacity planning.
- **Decisão:** DynamoDB on-demand para todas as tabelas. TTL nativo para expiração automática de eventos. Streams para export para S3.
- **Consequências:** Custo por request (pay-per-use). Modelo de dados desnormalizado. Sem queries ad-hoc complexas — analytics via Athena sobre S3.
- **Alternativas descartadas:** Aurora Serverless (custo mínimo mais alto, cold start), ElastiCache (volátil, sem persistência), MongoDB Atlas (custo + gerenciamento externo).

### ADR-005: Fan-out Multi-Canal via EventBridge + Cascata no Marketing Cloud

- **Contexto:** Geofence events precisam acionar múltiplos canais (Push, WhatsApp, Email, SMS). Disparar todos simultaneamente gera custo alto e experiência ruim pro usuário (mensagens duplicadas). A lógica de orquestração entre canais (ex: “se não abriu push em 10min, manda WhatsApp”) é complexa de implementar em Lambdas customizadas.
- **Decisão:** EventBridge faz fan-out para filas SQS por canal. Três estratégias configuráveis por regra: (1) `fanout` — todos os canais simultâneos; (2) `cascade` — canal primário imediato + fallback via Journey Builder do MC com delay; (3) `journey` — toda orquestração delegada ao MC Journey Builder.
- **Consequências:** Estratégia `cascade` reduz custo de WhatsApp em 60-80% (só fallback). Journey Builder do MC gerencia a lógica temporal, evitando implementação de state machines no backend. Trade-off: dependência do MC para orquestração temporal — mitigada por Firebase como canal primário independente.
- **Alternativas descartadas:** Step Functions para orquestração (latência alta, complexidade de manutenção, custo por transição de estado), lógica de delay em Lambda com DynamoDB Streams (complexidade operacional, difícil de debugar), SES + SNS direto (sem orquestração entre canais).

-----

## 08 — Segurança & Privacidade

### 8.1 Autenticação Mobile → API

JWT assinado no device com chave rotativa. API Gateway valida token via Lambda Authorizer. Rate limit por user_id: 10 req/min (proteção contra geofence bouncing).

### 8.2 Admin API

Endpoint separado com autenticação Cognito + MFA. Roles: admin (CRUD total), operator (read + ativar/desativar), viewer (read-only). Audit log de todas as mudanças.

### 8.3 Dados em Trânsito & Repouso

TLS 1.3 em todas as conexões. DynamoDB encryption at rest (AES-256) com AWS KMS. Dados de localização classificados como PII — acesso restrito via IAM policies.

### 8.4 LGPD / Privacidade

Consentimento explícito de geolocalização no app (permissão do SO + opt-in na feature). Dados de localização com TTL (90 dias). Endpoint de exclusão (DSAR) para direito ao esquecimento.

-----

## 09 — Observabilidade

|Pilar        |Ferramenta                |O Que Monitorar                                                                                         |Alertas                                                                   |
|-------------|--------------------------|--------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
|**Métricas** |CloudWatch Metrics        |Lambda duration/errors, API Gateway 4xx/5xx, DynamoDB throttles, SQS age of oldest message              |Lambda error rate > 1%, SQS age > 60s, DynamoDB throttle > 0              |
|**Logs**     |CloudWatch Logs + Insights|Structured JSON logs em cada Lambda. Campos: user_id, geofence_id, eligible, blocked_reason, duration_ms|Anomalia em volume de eventos blocked (> 95% sugere regra mal configurada)|
|**Traces**   |AWS X-Ray                 |Trace end-to-end: API Gateway → Processor → Eligibility → EventBridge → Connector → MC/Firebase         |p99 > 2s, trace com erro em qualquer segmento                             |
|**Dashboard**|CloudWatch Dashboard      |Eventos/min por geofence, taxa de elegibilidade, notificações enviadas vs. falhas, top blocked_reasons  |Drop > 50% no volume de eventos (possível bug no SDK mobile)              |
|**Business** |Athena + QuickSight       |Conversão pós-geofence, heatmap de visitas por geofence, análise de cooldown effectiveness              |—                                                                         |

### KPIs Recomendados

- **Taxa de elegibilidade** — eventos elegíveis / total de eventos
- **Taxa de entrega por canal** — notificações entregues / enviadas, segmentado por Push, WhatsApp, Email
- **Taxa de fallback** — % de eventos onde canal secundário (WhatsApp) foi acionado por falha/não-abertura do primário (Push)
- **Custo por engajamento** — custo total de canais / usuários que engajaram (abriram push, leram WA, clicaram email)
- **Tempo médio de processamento** — p50 e p95 do pipeline completo
- **Conversão pós-geofence** — ação no app dentro de 30min após notificação, segmentado por canal

-----

## 10 — Estratégia Multi-Canal

### 10.1 Visão Geral

O EventBridge recebe um único evento do Eligibility Engine e faz fan-out para quantos canais forem definidos na regra de elegibilidade. Cada canal opera de forma independente — se o WhatsApp falha, o Push não é afetado.

```
                          ┌──▶ SQS Push ────▶ Lambda Firebase Conn. ──▶ Firebase FCM
                          │
EventBridge ──fan-out──┬──┼──▶ SQS WhatsApp ▶ Lambda MC WA Conn. ───▶ MC Journey ──▶ WhatsApp Business API
                       │  │
                       │  └──▶ SQS Email ───▶ Lambda MC Email Conn. ▶ MC Journey ──▶ Email SMTP
                       │
                       └─────▶ SQS Analytics ▶ Lambda S3 ───────────▶ S3 (Parquet) ──▶ Athena
```

### 10.2 Estratégias de Orquestração

|Estratégia   |Comportamento                                                      |Quando Usar                                                                |Custo                            |
|-------------|-------------------------------------------------------------------|---------------------------------------------------------------------------|---------------------------------|
|**`fanout`** |Todos os canais disparados simultaneamente                         |Comunicações urgentes (ex: alerta de segurança, promoção relâmpago)        |Alto — todos os canais acionados |
|**`cascade`**|Canal primário imediato; fallback após delay se usuário não engajou|Campanhas de marketing (ex: push primeiro, WhatsApp se não abriu em 10min) |Médio — WhatsApp só como fallback|
|**`journey`**|Toda orquestração delegada ao Marketing Cloud Journey Builder      |Fluxos complexos com múltiplas etapas, A/B testing, personalização avançada|Variável — MC decide os canais   |

### 10.3 Configuração no Painel Admin

O time de CRM configura a estratégia diretamente na regra de elegibilidade, sem necessidade de código:

```json
{
  "notification_config": {
    "channels": ["push", "whatsapp"],
    "strategy": "cascade",
    "cascade_delay_minutes": 10,
    "channel_configs": {
      "push": {
        "connector": "firebase",
        "priority": 1
      },
      "whatsapp": {
        "connector": "marketing_cloud",
        "journey_key": "GEOFENCE_WA_VILALOBOS",
        "template_name": "geofence_welcome",
        "priority": 2
      }
    }
  }
}
```

### 10.4 Integração com Marketing Cloud Journey Builder

Para estratégias `cascade` e `journey`, o Lambda Connector envia um evento de entrada no Journey:

```json
POST /interaction/v1/events
{
  "ContactKey": "usr_abc123",
  "EventDefinitionKey": "GEOFENCE_ENTRY_WHATSAPP",
  "Data": {
    "geofence_name": "Shopping Vila Lobos",
    "geofence_id": "geo_042",
    "user_segment": "premium",
    "event_type": "ENTER",
    "timestamp": "2026-04-12T14:30:00Z",
    "push_delivered": true,
    "strategy": "cascade"
  }
}
```

O Journey Builder do Marketing Cloud então orquestra internamente: aguarda o delay configurado, verifica engajamento (abertura do push via callback), e decide se aciona o canal seguinte (WhatsApp, Email, SMS).

### 10.5 Adicionando Novos Canais

Adicionar um canal novo (ex: RCS, SMS, in-app message) requer apenas:

1. Nova regra no EventBridge (filtro por conteúdo do evento)
1. Nova fila SQS dedicada ao canal
1. Nova Lambda Connector que integra com a API do canal
1. Atualizar o painel admin para incluir o canal nas opções

O restante da arquitetura (device, Eligibility Engine, DynamoDB) não muda.

-----

*DAS — Plataforma de Geofencing Inteligente · v1.1.0 · Abril 2026*
