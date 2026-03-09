# JFBot

Bot de atendimento via WhatsApp com NestJS + processamento assíncrono com BullMQ/Redis e classificação por IA.

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
- Redis (ioredis)
- BullMQ
- pnpm

## Tomada de decisões em Regras de negócio

- Eu, como desenvolvedor, fui obrigado a tomar algumas regras de negócio como padrão por ter pensado este bot para "Emergências e SOS".
  Apliquei regra de duração máxima de atendimento para 5 minutos, a fim de tornar eficiente em caso de socorro ou risco de vida.
  Alguns fluxos foram fixados para não utilização de IA e os que utilizam, utilizam seu próprio prompt. Isso torna o "atendimento" mais eficiente e dinâmico por áreas de atuação.
- (Se houver outros casos como este, podem descrever e gerar a PR para atualização.)

## Arquitetura Ativa

`AppModule` atual importa:

- `src/bot/whatsapp.module.ts`
- `src/integracao-api/integracao-api.module.ts`

Fluxo principal:

1. WhatsApp (socket)
2. `src/bot/whatsapp.service.ts` (orquestra conversa)
3. `incident-queue` (BullMQ)
4. `src/queue/incident.worker.ts`
5. IA (`src/groq/groq.service.ts`)
6. Deduplicação Redis (`src/dedup/dedup.service.ts`)
7. Resposta ao usuário + integração externa

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
- `src/integracao-api/integracao-api.service.ts`: envio do payload de ocorrência para API externa
- `src/storage/auth.store.ts`: persistência de sessão em `./auth/auth.json`
- `src/utils/logger.ts`: logs padronizados

## Requisitos

- Node.js 20+
- pnpm 10+
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

## Contribuição

- Consulte [CONTRIBUTING.md](/home/phdiniz/projects/wbjf-api/CONTRIBUTING.md) para abrir issues e enviar PRs.
- Push direto em `main` deve permanecer bloqueado; merge em `main` ocorre via PR com revisão.

## Deploy (GitHub Actions)

Workflow: `.github/workflows/deploy.yml`

- gatilhos: `push` em `main` e `workflow_dispatch`
- etapas:
  - build no GitHub Actions (`pnpm install --frozen-lockfile` + `pnpm run build`)
  - sincronização dos arquivos para o servidor via `rsync` sobre SSH
  - execução remota de comandos via SSH (install/build/restart)

Secrets obrigatórios:

- `SSH_HOST`
- `SSH_USER`
- `SSH_PRIVATE_KEY`
- `DEPLOY_PATH`

Secrets opcionais:

- `SSH_PORT` (default `22`)
- `REMOTE_INSTALL_CMD` (default `pnpm install --frozen-lockfile --prod=false`)
- `REMOTE_BUILD_CMD` (default `pnpm run build`)
- `REMOTE_RESTART_CMD` (default `echo 'No restart command configured'`)

### Como gerar e configurar a chave SSH (servidor + GitHub)

1. No servidor, gere um par de chaves para o deploy:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/github_actions_deploy -N ""
```

2. Autorize a chave pública para o usuário que receberá o deploy:

```bash
cat ~/.ssh/github_actions_deploy.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

3. Copie a chave privada para cadastrar no GitHub:

```bash
cat ~/.ssh/github_actions_deploy
```

4. No GitHub do repositório:
   - abra `Settings` -> `Secrets and variables` -> `Actions`
   - clique em `New repository secret`
   - crie `SSH_PRIVATE_KEY` colando o conteúdo completo da chave privada
   - crie também:
     - `SSH_HOST` (IP ou domínio do servidor)
     - `SSH_USER` (usuário SSH no servidor)
     - `DEPLOY_PATH` (pasta de destino no servidor)
     - `SSH_PORT` (opcional, se diferente de `22`)

5. Valide a conexão manualmente antes do primeiro deploy:

```bash
ssh -i ~/.ssh/github_actions_deploy <SSH_USER>@<SSH_HOST> -p <SSH_PORT>
```

## Swagger (OpenAPI)

Swagger está habilitado no bootstrap do NestJS.

- URL padrão local: `http://localhost:3000/docs`
- caminho configurável por variável: `SWAGGER_PATH`

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
GROQ_CONVERSATIONAL_TEMPERATURE=0.55
GROQ_CONVERSATIONAL_MAX_COMPLETION_TOKENS=320
GROQ_HUMAN_FOLLOWUP_MAX_COMPLETION_TOKENS=220
GROQ_GENERIC_FOLLOWUP_MAX_COMPLETION_TOKENS=220
FOLLOWUP_HISTORY_MESSAGES=12
CONVERSATION_INACTIVITY_TIMEOUT_MINUTES=15
SERVER_IDLE_RESTART_MINUTES=0
REVERSE_GEOCODE_CACHE_TTL_MS=600000

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_PREFIX=pjf
REDIS_DISABLED=false
WA_LOCAL_OUTBOX_FILE=var/incident-outbox.json
WA_LOCAL_OUTBOX_RETRY_INTERVAL_MS=15000

SWAGGER_PATH=docs
BACKEND_PUBLIC_URL="https://abcd-ec2.exemplo.com.br"
SOS_JF_ENDPOINT_URL="http://sos-jf.ddns.net/v1/wbjf/"
INTEGRACAO_API_KEY=
```

Referência das variáveis de IA e contexto:

| Variável                                      |                     Padrão | Uso                                                                                             |
| --------------------------------------------- | -------------------------: | ----------------------------------------------------------------------------------------------- |
| `GROQ_MAX_COMPLETION_TOKENS`                  |                      `800` | Limite de tokens para classificação JSON (`generateCategorizedJson`).                           |
| `GROQ_CONVERSATIONAL_TEMPERATURE`             |                     `0.55` | Criatividade das mensagens conversacionais (`request_details`, `general_guidance`, follow-ups). |
| `GROQ_CONVERSATIONAL_MAX_COMPLETION_TOKENS`   |                      `320` | Limite de tokens das mensagens conversacionais.                                                 |
| `GROQ_HUMAN_FOLLOWUP_MAX_COMPLETION_TOKENS`   |                      `220` | Limite de tokens da confirmação humanizada no fluxo categorizado.                               |
| `GROQ_GENERIC_FOLLOWUP_MAX_COMPLETION_TOKENS` |                      `220` | Limite de tokens da confirmação humanizada no fluxo genérico de continuidade.                   |
| `FOLLOWUP_HISTORY_MESSAGES`                   |                       `12` | Quantidade de mensagens recentes enviada como contexto no follow-up.                            |
| `CONVERSATION_INACTIVITY_TIMEOUT_MINUTES`     |                        `5` | Tempo de inatividade (minutos) para encerrar a conversa automaticamente.                        |
| `SERVER_IDLE_RESTART_MINUTES`                 |                        `0` | Reinício automático após inatividade do servidor (em minutos). `0` ou vazio desativa.           |
| `REVERSE_GEOCODE_CACHE_TTL_MS`                |                   `600000` | TTL do cache de reverse geocode (em ms) para coordenadas repetidas.                             |
| `WA_LOCAL_OUTBOX_FILE`                        | `var/incident-outbox.json` | Arquivo local durável para fallback de jobs quando enqueue no Redis/fila falhar.                |
| `WA_LOCAL_OUTBOX_RETRY_INTERVAL_MS`           |                    `15000` | Intervalo de retentativa (ms) para reenfileirar jobs salvos no outbox local.                    |
| `BACKEND_PUBLIC_URL`                          |                        `-` | Base pública usada para montar `linkDaMidia` no formato `/public/assets/media/{media_hash}`.    |
| `SOS_JF_ENDPOINT_URL`                         |                        `-` | Endpoint principal de integração externa (`POST`).                                              |
| `INTEGRACAO_ENDPOINT_URL`                     |                        `-` | Fallback legado de endpoint quando `SOS_JF_ENDPOINT_URL` não estiver definido.                  |
| `INTEGRACAO_API_KEY`                          |                        `-` | Chave enviada no header HTTP `X-API-KEY` da integração externa.                                 |

### Regras Redis

- `REDIS_PREFIX` é opcional.
- Comportamento atual:
  - se `REDIS_PREFIX` não existir: usa padrão `pjf`
  - se `REDIS_PREFIX` for vazio: desativa prefixo de chaves Redis
- `REDIS_DISABLED=true` força modo totalmente em memória (sem uso de Redis).
- `REDIS_PASSWORD` é obrigatório quando `REDIS_HOST` não é `localhost` nem `127.0.0.1`.
- Log de sucesso da conexão:
  - `[REDIS] Connected host=... port=...`
- Se Redis ficar offline durante uma conversa:
  - aquela conversa entra em modo offline (memória)
  - ao retornar o Redis, as alterações pendentes dessa conversa são sincronizadas automaticamente (set/del com TTL)

## Fluxo conversacional (WhatsApp)

### 1) Primeira interação

O usuário recebe o template:

- saudação com primeiro nome
- aviso de SLA conversacional: `Esta solicitação terá duração máxima de 5 minutos`
- opções:
  - `1 Defesa Civil` (resgate e áreas de risco)
  - `2 Direitos Humanos` (desaparecimento e violações de direitos)
  - `3 Desenvolvimento Social` (falta de alimentos e vulnerabilidade social)
  - `4 Assistência Social` (famílias desalojadas e proteção social imediata)
  - `5 EMCASA` (moradia/habitação e casa destruída)
  - `6 Defesa Animal` (maus-tratos e animais em risco)
  - `7 Canil Municipal` (recolhimento e manejo animal)
  - `8 Procon` (defesa do consumidor e denúncias de preço)
  - `9 Secretaria de Comunicação` (informações públicas e orientação oficial)
  - `0 Encerrar`

Comando adicional:

- `menu`: reabre o menu inicial em qualquer etapa do atendimento.

### 2) Encerramento manual

Comandos aceitos a qualquer momento:

- `encerrar`
- `0`

Resposta:

- `Agradecemos pelo contato. Se precisar de algo, entre em contato.`

### 3) Opções 1-9

Após escolher `1` a `9`, o bot responde:

- `Descreva com detalhes sua solicitação`

Em seguida entra na etapa de coleta obrigatória.

### 4) Coleta obrigatória (1-9)

Antes de enviar para IA, exige:

- Endereço completo (com número aproximado)
- Vítimas (`Sim/Não/Não sei`)
- Imagem ou vídeo obrigatório

Regras:

- Se o endereço já vier na descrição inicial, não solicita novamente esse campo.
- Se o usuário enviar coordenadas (texto/link/localização do WhatsApp), o bot resolve para endereço aproximado.
- Se faltar algo, retorna apenas pendências faltantes em uma única mensagem (sem quebrar em múltiplos chunks).
- Template atual de pendências:
  - `Estou com você e já vou dar sequência 🙏`
  - `Para continuar, preciso destes dados agora. Ainda faltam:`
  - `- Endereço completo (com número aproximado)`
  - `ou sua Localização do WhatsApp.`
  - `- Vítimas? Sim/Não/Não sei`
  - `- Enviar imagem ou vídeo (obrigatório) aqui no chat mesmo.`
- Mídia é salva em `./public/assets/media/{uuid}.{ext}`.
- Se o usuário enviar apenas mídia (sem texto), a mídia é aceita e o bot solicita somente os campos restantes.
- Quando o endereço estiver pendente apenas por número, o bot envia:
  - `" {endereço atual/base} "`
  - `Se o endereço estiver correto, informe apenas o número correto do endereço.`
- `endereco_completo` final é normalizado para endereço + número (quando houver base de endereço); número isolado não é aceito como endereço completo.
- Mapeamento de vítimas para `pessoas_afetadas`:
  - `Sim` = 1
  - `Não` = 0
  - `Não sei` = 1

### 5) Inatividade

Se houver inatividade de 5 minutos:

- `Encerramos sua solicitação devido à inatividade de 5 minutos. Pode continuar por aqui normalmente.`
- o estado da conversa é encerrado e volta para `awaiting_option`
- para retomar, o usuário deve enviar `menu` ou uma opção `1..9`
- se a primeira mensagem após expiração já for `1..9`, o bot reaproveita a opção na mesma mensagem (sem exigir reenvio)

### 6) Follow-up pós-atendimento

Após cada atendimento concluído, o worker envia:

1. JSON da IA (com metadados operacionais)
2. Mensagem humanizada
3. `Podemos ajudar em algo mais?`

Interpretação de resposta:

- intenção positiva (`sim`, `claro`, `pode`, `continuar`, `continue`, `continua`, `prossiga`, `prosseguir`, `prossegue`, `seguir`, `siga`) -> abre um novo ciclo e reenvia o menu inicial
- intenção de encerramento (`não`, `nao`, `encerrar`, `0`, `obrigado`, `valeu` e variações) -> encerra
- outros textos -> prompt curto humanizado pedindo confirmação (`sim`/`não`)

## Fila e worker

Fila:

- nome: `incident-queue`
- tentativas por job: `3`
- backoff: exponencial (`2s`)
- worker concurrency: `5`
- timeout de enqueue no produtor (WhatsApp): `10s`
- fallback local persistente quando Redis/fila estiver indisponível:
  - arquivo padrão: `var/incident-outbox.json`
  - reenvio automático em background: a cada `15s` (padrão)
  - ao falhar enqueue, o job é salvo localmente e reenviado quando a fila voltar

Variáveis do outbox local:

- `WA_LOCAL_OUTBOX_FILE` (default: `var/incident-outbox.json`)
- `WA_LOCAL_OUTBOX_RETRY_INTERVAL_MS` (default: `15000`)

Tipos de job (`IncidentJobData`):

- `categorized`
  - usado para menu 1-9
  - inclui `service` de contexto selecionado no menu
  - inclui `enderecoCompleto` para metadados finais e integração externa
  - envia payload para integração externa (`durch_api`)
- `followup_generic`
  - usado no fluxo de continuidade
  - sem `service`
  - não envia para integração externa

## IA

Parâmetros configuráveis de contexto/resposta:

- `GROQ_MAX_COMPLETION_TOKENS`: limite geral de tokens da classificação JSON.
- `GROQ_CONVERSATIONAL_TEMPERATURE`: temperatura das mensagens conversacionais.
- `GROQ_CONVERSATIONAL_MAX_COMPLETION_TOKENS`: limite de tokens para mensagens conversacionais.
- `GROQ_HUMAN_FOLLOWUP_MAX_COMPLETION_TOKENS`: limite de tokens da mensagem humanizada pós-triagem categorizada.
- `GROQ_GENERIC_FOLLOWUP_MAX_COMPLETION_TOKENS`: limite de tokens da mensagem humanizada no follow-up genérico.
- `FOLLOWUP_HISTORY_MESSAGES`: quantidade de mensagens recentes enviada ao modelo no fluxo de continuidade.

### Fluxo categorizado (menu 1-9)

Método: `generateCategorizedJson({ service, message })`

- Prompt base + orientação por serviço
- `orgao_responsavel` é definido pela categoria classificada (determinístico, conforme mapeamento do Mermaid)
- JSON final inclui:
  - `protocolo_atendimento` (numérico, 10 dígitos, não repetido em memória do worker)
  - `media_hash` (SHA-256 da mídia persistida com extensão, ex.: `{hash}.{ext}`)
  - `endereco_completo` (endereço do usuário já com número, quando disponível)
- valida JSON e normaliza schema
- fallback controlado em caso de erro/JSON inválido

Schema esperado:

```json
{
  "categoria": "string",
  "prioridade": "CRITICA|ALTA|MEDIA|BAIXA",
  "bairro": "string",
  "orgao_responsavel": "Defesa Civil|Direitos Humanos|Desenvolvimento Social|Assistência Social|EMCASA|Defesa Animal|Canil Municipal|Procon|Secretaria de Comunicação",
  "descricao_resumida": "string",
  "pessoas_afetadas": 1,
  "animais_afetados": 0,
  "risco_imediato": false,
  "protocolo_atendimento": "1234567890",
  "media_hash": "abc123...def456.jpg",
  "endereco_completo": "Rua Exemplo, 123, Bairro Exemplo"
}
```

Observação de classificação:

- O mapeamento de `orgao_responsavel` é determinístico pelo prompt/código.
- Atualmente não existe `Polícia Militar` no contrato de tipos/categorias.
- Mensagens como `roubaram minha casa` tendem a cair em `OUTROS`, portanto `Secretaria de Comunicação`.
- Se houver decisão de negócio para encaminhar segurança pública, será necessário alterar categoria/tipos/prompt e integração.

### Fluxo genérico (follow-up)

Método: `generateReply(message)`

- classificador genérico com mapeamento de categorias no prompt
- normalização de campos obrigatórios
- fallback textual em falha
- quando o usuário retorna com novas informações, o worker envia para a IA a mensagem atual + histórico recente de mensagens do usuário (`conversations/{telefone}.json`) para manter contexto

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

- envio HTTP real habilitado
- `sendAiResponse(...)` faz `POST` para `SOS_JF_ENDPOINT_URL` (ou `INTEGRACAO_ENDPOINT_URL` como fallback legado)

Momento do envio:

- imediatamente após o JSON da IA ser enviado no WhatsApp, apenas para fluxo `categorized`

Payload inclui:

- `phone`
- `ocorrencia`
- `endereco_completo` (quando presente no JSON da IA ou no input do worker)
- `linkDaMidia` (quando houver mídia)
- `payloadIa` (JSON da IA serializado em string formatada)

Observações:

- Header de autenticação: `X-API-KEY` (valor de `INTEGRACAO_API_KEY`).
- URL padrão de fallback no código: `http://sos-jf.ddns.net/v1/wbjf/`.
- `linkDaMidia` usa `BACKEND_PUBLIC_URL` + `/public/assets/media/{media_hash}` quando `media_hash` existir no `payloadIa`.
- enriquecimento de metadados é resiliente: falha ao gerar `media_hash` não remove `protocolo_atendimento` nem `endereco_completo`.

## Persistência local

- Sessão/auth do WhatsApp: `./auth/auth.json`
- Mídias recebidas: `./public/assets/media/`
- Histórico de conversa por telefone: `./conversations/{telefone}.json`
  - inclui mensagens de entrada (`role: "user"`) e saída (`role: "bot"`)
  - cada arquivo contém timestamps para auditoria do fluxo com IA/Bot

## Logs operacionais esperados

- `[REDIS] Connected ...`
- `[WA] Connected`
- `[QUEUE] message added`
- `[QUEUE] job received`
- `[AI] classified`
- `[DEDUP] updated reports`
- `[WA] media detected`
- `[WA] media persisted`
- `[QUEUE] failed to generate media_hash` (somente quando não for possível ler o arquivo de mídia)
- `[QUEUE] failed to append metadata to ai json` (somente em falha de hash/metadado)
- `[WA] response sent`
- `[INTEGRACAO_API] payload sent`

Observação operacional:

- o bot processa `messages.upsert` dos tipos `notify` e `append`
- isso evita perda de resposta após logs como `Timeout in AwaitingInitialSync, forcing state to Online and flushing buffer`
- sem interação de usuários por 30 minutos, o processo encerra com `exit 0` para reinício automático via supervisor.

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

## Testes de regressão

Foram adicionados testes de regressão para os fluxos críticos recentes:

- `src/bot/conversation-session.service.regression.spec.ts`
  - fallback em memória quando Redis fica indisponível
  - sincronização de pendências para Redis após recuperação
  - compatibilidade com estado legado de coleta obrigatória
- `src/bot/whatsapp.service.regression.spec.ts`
  - parsing de opções `0-9`
  - comando `menu`
  - interpretação de vítimas (`sem vítimas` / `com vítimas`)
  - template inicial com aviso de 5 minutos
  - número isolado não é aceito como endereço completo
  - `endereco_completo` é montado como base + número quando necessário
- `src/groq/groq.service.regression.spec.ts`
  - mensagem de dados faltantes em template único padronizado
  - ausência de instrução de mídia quando mídia não está pendente
  - mensagem de inatividade em 5 minutos
- `src/queue/incident.worker.regression.spec.ts`
  - protocolo numérico de 10 dígitos sem repetição imediata
  - inclusão de `protocolo_atendimento`, `media_hash` e `endereco_completo` no JSON final

### Qualidade de código (lint)

Lint validado para os arquivos críticos de fluxo e regressão:

- `src/bot/conversation-session.service.regression.spec.ts`
- `src/bot/whatsapp.service.regression.spec.ts`
- `src/bot/whatsapp.service.ts`
- `src/groq/groq.service.regression.spec.ts`
- `src/main.ts`
- `src/queue/incident.worker.regression.spec.ts`
- `src/storage/auth.store.ts`

Comando utilizado:

```bash
pnpm -s eslint src/bot/conversation-session.service.regression.spec.ts src/bot/whatsapp.service.regression.spec.ts src/bot/whatsapp.service.ts src/groq/groq.service.regression.spec.ts src/main.ts src/queue/incident.worker.regression.spec.ts src/storage/auth.store.ts
```

## Estado atual das opções do menu

- `1 Defesa Civil`: ativo com IA + coleta obrigatória
- `2 Direitos Humanos`: ativo com IA + coleta obrigatória
- `3 Desenvolvimento Social`: ativo com IA + coleta obrigatória
- `4 Assistência Social`: ativo com IA + coleta obrigatória
- `5 EMCASA`: ativo com IA + coleta obrigatória
- `6 Defesa Animal`: ativo com IA + coleta obrigatória
- `7 Canil Municipal`: ativo com IA + coleta obrigatória
- `8 Procon`: ativo com IA + coleta obrigatória
- `9 Secretaria de Comunicação`: ativo com IA + coleta obrigatória
- `0 Encerrar`: encerra conversa imediatamente

## Observação sobre o código 🔐

Existe um repositório diferente contendo o código fonte completo do projeto desenvolvido em NestJS. Por questões de segurança e privacidade, este código não estará disponível para visualização. Issues devem ser reportadas neste repositório.
