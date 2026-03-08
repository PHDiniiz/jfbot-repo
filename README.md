# JFBot

Bot de atendimento via WhatsApp com NestJS + Baileys, processamento assíncrono com BullMQ/Redis e classificação por IA (Groq).

## Objetivo

Este projeto recebe mensagens do WhatsApp, aplica um fluxo conversacional de triagem, envia incidentes para processamento assíncrono e responde o usuário com:

1. JSON estruturado da IA
2. Mensagem humanizada
3. Pergunta de continuidade: `Podemos ajudar em algo mais?`

Sem banco de dados relacional no momento. O estado operacional é persistido em Redis e em arquivos locais para sessão do WhatsApp.

## Stack

- Node.js
- NestJS
- TypeScript
- Baileys
- Redis (ioredis)
- BullMQ
- Groq SDK
- pnpm

## Arquitetura Ativa

`AppModule` atual importa:

- `src/bot/whatsapp.module.ts`
- `src/integracao-api/integracao-api.module.ts`

Fluxo principal:

1. WhatsApp (Baileys socket)
2. `src/bot/whatsapp.service.ts` (orquestra conversa)
3. `incident-queue` (BullMQ)
4. `src/queue/incident.worker.ts`
5. IA (`src/groq/groq.service.ts`)
6. Deduplicação Redis (`src/dedup/dedup.service.ts`)
7. Resposta ao usuário + preview de integração externa

## Estrutura Relevante

- `src/bot/`
  - `whatsapp.service.ts`: socket, parsing, estados conversacionais e enfileiramento
  - `conversation-session.service.ts`: estados Redis (menu, coleta obrigatória, follow-up, timeout)
- `src/queue/`
  - `redis.connection.ts`: conexão Redis, prefixo de chave e configuração BullMQ
  - `incident.queue.ts`: contrato de job e fila
  - `incident.worker.ts`: processamento assíncrono
- `src/groq/groq.service.ts`: geração de JSON e mensagens humanizadas
- `src/dedup/dedup.service.ts`: agregação por `categoria+bairro` com TTL
- `src/integracao-api/integracao-api.service.ts`: preview do JSON que iria para API externa
- `src/storage/auth.store.ts`: persistência de sessão Baileys em `./auth/auth.json`
- `src/utils/logger.ts`: logs padronizados

## Requisitos

- Node.js 20+
- pnpm 9+
- Redis acessível

## Instalação e execução

```bash
pnpm install
pnpm start
```

Desenvolvimento:

```bash
pnpm dev
```

Build de produção:

```bash
pnpm build
pnpm start:prod
```

## Variáveis de ambiente

Carregamento:

- `main.ts` chama `loadEnvironmentVariables()`
- ordem de leitura: `.env.development` (ou `.env.production`) e depois `.env`

Exemplo mínimo (`.env.development`):

```env
NODE_ENV=development
PORT=3000

GROQ_API_KEY=
GROQ_MODEL=openai/gpt-oss-120b
GROQ_MAX_COMPLETION_TOKENS=800

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_PREFIX=pjf

INTEGRACAO_API__URL=http://sos-jf.ddns.net/api/
INTEGRACAO_API__X_API_KEY=
```

### Regras Redis

- `REDIS_PREFIX` é opcional.
- Comportamento atual:
  - se `REDIS_PREFIX` não existir: usa padrão `pjf`
  - se `REDIS_PREFIX` for vazio: desativa prefixo de chaves Redis
- `REDIS_PASSWORD` é obrigatório quando `REDIS_HOST` não é `localhost` nem `127.0.0.1`.
- Log de sucesso da conexão:
  - `[REDIS] Connected host=... port=...`

## Fluxo conversacional (WhatsApp)

### 1) Primeira interação

O usuário recebe o template:

- saudação com primeiro nome
- opções:
  - `1 Defesa Civil`
  - `2 Bombeiros`
  - `3 Polícia Militar`
  - `0 Encerrar`

### 2) Encerramento manual

Comandos aceitos a qualquer momento:

- `encerrar`
- `0`

Resposta:

- `Agradecemos pelo contato. Se precisar de algo, entre em contato.`

### 3) Opções 1-3

Após escolher `1`, `2` ou `3`, o bot responde:

- `Descreva com detalhes sua solicitação`

Em seguida entra na etapa de coleta obrigatória.

### 4) Coleta obrigatória (1-3)

Antes de enviar para IA, exige:

- Endereço completo
- Vítimas (`Sim/Não/Não sei`)
- Imagem ou vídeo obrigatório

Regras:

- Se o endereço já vier na descrição inicial, não solicita novamente esse campo.
- Se faltar algo, retorna apenas pendências faltantes.
- Mídia é salva em `./public/assets/media/{uuid}.{ext}`.
- Mapeamento de vítimas para `pessoas_afetadas`:
  - `Sim` = 1
  - `Não` = 0
  - `Não sei` = 1

### 5) Inatividade

Se houver inatividade de 15 minutos, o usuário é avisado:

- `Encerramos sua conversa por 15 minutos de inatividade. Pode continuar por aqui normalmente.`

### 6) Follow-up pós-atendimento

Após cada atendimento concluído, o worker envia:

1. JSON da IA
2. Mensagem humanizada
3. `Podemos ajudar em algo mais?`

Interpretação de resposta:

- intenção positiva (`sim`, `claro`, `pode`, `continuar`, `prosseguir`) -> `Descreva sua solicitação...`
- intenção de encerramento (`não`, `nao`, `encerrar`, `0`, `obrigado`, `valeu` e variações) -> encerra
- outros textos -> prompt curto humanizado pedindo confirmação (`sim`/`não`)

## Fila e worker

Fila:

- nome: `incident-queue`
- tentativas por job: `3`
- backoff: exponencial (`2s`)
- worker concurrency: `5`

Tipos de job (`IncidentJobData`):

- `categorized`
  - usado para menu 1-3
  - inclui `service` fixo
- `followup_generic`
  - usado no fluxo de continuidade
  - sem `service`

## IA (Groq)

### Fluxo categorizado (menu 1-3)

Método: `generateCategorizedJson({ service, message })`

- Prompt base + orientação por serviço
- `orgao_responsavel` é fixado pelo serviço escolhido (determinístico)
- valida JSON e normaliza schema
- fallback controlado em caso de erro/JSON inválido

Schema esperado:

```json
{
  "categoria": "string",
  "prioridade": "CRITICA|ALTA|MEDIA|BAIXA",
  "bairro": "string",
  "orgao_responsavel": "Defesa Civil|Bombeiros|Polícia Militar",
  "descricao_resumida": "string",
  "pessoas_afetadas": 1,
  "animais_afetados": 0,
  "risco_imediato": false
}
```

### Fluxo genérico (follow-up)

Método: `generateReply(message)`

- classificador genérico com mapeamento de categorias no prompt
- normalização de campos obrigatórios
- fallback textual em falha

### Mensagens humanizadas

- `generateHumanFollowUpMessage(...)` para fluxo categorizado
- `generateGenericHumanFollowUpMessage(aiJsonResponse)` para follow-up
  - prioriza explicar nível de prioridade de forma acolhedora
  - fallback considera prioridade e órgão extraídos do JSON

### Intenção de abrigo no follow-up

No worker, fluxo `followup_generic` detecta termos de abrigo e usa:

- `generateNearestShelterReply(...)`

## Deduplicação em Redis

Serviço: `src/dedup/dedup.service.ts`

- chave: `incident:{categoria}:{bairro}` com prefixo opcional
- operação: `INCR`
- TTL: `15 minutos`

Exemplo de chave real com prefixo padrão:

- `pjf:incident:AREA_RISCO:Benfica`

## Integração externa (INTEGRACAO_API)

Serviço: `src/integracao-api/integracao-api.service.ts`

Status atual:

- envio HTTP real está depreciado
- `sendAiResponse(...)` atualmente faz preview e `console.log` do JSON final

Log gerado:

- `[INTEGRACAO_API][PREVIEW_JSON] { ... }`

Payload inclui:

- `phone`
- `nome_contato` (primeiro nome sem acento)
- `service` (quando categorizado)
- `aiResponse` (mesmo JSON enviado ao usuário)
- `mediaPath`
- `timestamp`

## Persistência local

- Sessão/auth do WhatsApp: `./auth/auth.json`
- Mídias recebidas: `./public/assets/media/`

## Logs operacionais esperados

- `[REDIS] Connected ...`
- `[WA] Connected`
- `[QUEUE] message added`
- `[QUEUE] job received`
- `[AI] classified`
- `[DEDUP] updated reports`
- `[WA] response sent`
- `[INTEGRACAO_API][PREVIEW_JSON] ...`

## Troubleshooting

### `ECONNREFUSED 127.0.0.1:6379`

Causa comum:

- variáveis `REDIS_*` ausentes ou Redis fora do ar

Checklist:

1. validar `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`
2. confirmar que `.env.development` está na raiz do projeto
3. confirmar startup com log `[REDIS] Connected ...`

### Bot não responde

Checklist:

1. verificar estado da conexão WhatsApp (`[WA] Connected`)
2. verificar se a mensagem entrou na fila (`[QUEUE] message added`)
3. verificar processamento (`[QUEUE] job received`)
4. verificar falhas no worker (`[QUEUE] job failed`)

### Resposta duplicada

Há proteção de deduplicação de eventos recebidos por `messageId` em memória no `WhatsAppService`.

## Estado atual das opções do menu

- `1 Defesa Civil`: ativo com IA + coleta obrigatória
- `2 Bombeiros`: ativo com IA + coleta obrigatória
- `3 Polícia Militar`: ativo com IA + coleta obrigatória
- `4-6`: atualmente não disponíveis no fluxo ativo

## Observação sobre código legado

Existe uma implementação antiga em `src/whatsapp/*` (gateway e serviço antigos). O fluxo em produção atual está em `src/bot/*`, que é o módulo importado pelo `AppModule`.
