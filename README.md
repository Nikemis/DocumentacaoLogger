# Logger — PZaaS

Serviço de observabilidade do projeto **PZaaS (Pizza as a Service)**. Recebe logs estruturados de qualquer outro serviço da pizzaria, guarda no Redis e permite consultá-los — inclusive filtrando por pedido.

> URL pública deste serviço: `https://pzaas.online/webhook/v1/logs-240285-239685` (produção) — todos os exemplos abaixo já usam essa URL real.

## Convenções (herdadas do contrato global da turma)

- Versionamento: `/v1`
- Protocolo: HTTP / JSON
- Header obrigatório em todas as chamadas: `Content-Type: application/json`
- Header obrigatório de autenticação: `x-api-key: turma2026`
- Header `x-pedido-id`: opcional — quando presente na gravação, o log fica associado ao pedido e pode ser filtrado depois
- `GET /health` não exige `x-api-key` (para permitir monitoramento externo sem credencial)

## Endpoints

### `POST /v1/logs`

Registra um log estruturado.

**Headers**

| Header | Obrigatório | Descrição |
|---|---|---|
| `Content-Type` | Sim | `application/json` |
| `x-api-key` | Sim | `turma2026` |
| `x-pedido-id` | Não | ID do pedido, se este log pertence a um fluxo de pedido já iniciado |

**Body**

```json
{
  "service": "forno",
  "level": "info",
  "message": "Pedido 4821 entrou no forno",
  "timestamp": "2026-09-05T18:30:00.000Z",
  "meta": { "tempo_estimado_min": 12 }
}
```

| Campo | Obrigatório | Tipo | Descrição |
|---|---|---|---|
| `service` | Sim | string | Nome do serviço que originou o log (ex.: `forno`, `pagamento`) |
| `level` | Sim | string | Um de: `debug`, `info`, `warn`, `error` |
| `message` | Sim | string | Mensagem do log |
| `timestamp` | Não | string (ISO 8601) | Se omitido, usa o horário de recebimento |
| `meta` | Não | object | Qualquer dado extra relevante |

**Resposta de sucesso — `201`**

```json
{
  "status": "ok",
  "message": "log registrado com sucesso",
  "pedido_id": "4821"
}
```

**Exemplo real de chamada**

```bash
curl -X POST "https://pzaas.online/webhook/v1/logs-240285-239685" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: 4821" \
  -d '{"service":"forno","level":"info","message":"Pedido 4821 entrou no forno"}'
```

```json
{ "status": "ok", "message": "log registrado com sucesso", "pedido_id": "4821" }
```

### `PUT /v1/logs`

Substitui **todos** os logs de um pedido por uma nova lista. Usado quando é preciso reescrever o histórico completo de um pedido (ex.: correção em lote).

**Headers**

| Header | Obrigatório | Descrição |
|---|---|---|
| `Content-Type` | Sim | `application/json` |
| `x-api-key` | Sim | `turma2026` |
| `x-pedido-id` | Sim | ID do pedido cujos logs serão substituídos |

**Body**

```json
{
  "logs": [
    { "service": "forno", "level": "info", "message": "Pedido 4821 entrou no forno" },
    { "service": "fila-producao", "level": "info", "message": "Pedido 4821 liberado da fila" }
  ]
}
```

Cada item de `logs` segue as mesmas regras de `service`, `level` e `message` do `POST /v1/logs`.

**Resposta de sucesso — `200`**

```json
{ "status": "ok", "message": "logs do pedido substituídos com sucesso", "pedido_id": "4821", "total": 2 }
```

**Exemplo real de chamada**

```bash
curl -X PUT "https://pzaas.online/webhook/v1/logs-240285-239685" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: 4821" \
  -d '{"logs":[{"service":"forno","level":"info","message":"Pedido 4821 entrou no forno"}]}'
```

> Só substitui a lista específica do pedido (`pzaas:logs:<pedido_id>`). O histórico global (`pzaas:logs:all`) não é alterado — é tratado como registro imutável.

### `PATCH /v1/logs`

Atualiza parcialmente o **último** log registrado de um pedido (ex.: corrigir o `level` ou completar o `meta` depois do fato).

**Headers**

| Header | Obrigatório | Descrição |
|---|---|---|
| `Content-Type` | Sim | `application/json` |
| `x-api-key` | Sim | `turma2026` |
| `x-pedido-id` | Sim | ID do pedido cujo último log será atualizado |

**Body** — envie só os campos que quer alterar (`level`, `message` e/ou `meta`)

```json
{ "meta": { "tempo_estimado_min": 10 } }
```

**Resposta de sucesso — `200`**

```json
{
  "status": "ok",
  "message": "log atualizado com sucesso",
  "pedido_id": "4821",
  "log": {
    "timestamp": "2026-09-05T18:30:00.000Z",
    "received_at": "2026-09-05T18:30:00.120Z",
    "service": "forno",
    "level": "info",
    "message": "Pedido 4821 entrou no forno",
    "pedido_id": "4821",
    "meta": { "tempo_estimado_min": 10 }
  }
}
```

**Exemplo real de chamada**

```bash
curl -X PATCH "https://pzaas.online/webhook/v1/logs-240285-239685" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: 4821" \
  -d '{"meta":{"tempo_estimado_min":10}}'
```

Retorna `404` se o pedido não tiver nenhum log registrado.

### `DELETE /v1/logs`

Remove **todos** os logs de um pedido (a lista `pzaas:logs:<pedido_id>` inteira).

**Headers**

| Header | Obrigatório | Descrição |
|---|---|---|
| `x-api-key` | Sim | `turma2026` |
| `x-pedido-id` | Sim | ID do pedido cujos logs serão removidos |

**Resposta de sucesso — `200`**

```json
{ "status": "ok", "message": "logs do pedido removidos com sucesso", "pedido_id": "4821" }
```

**Exemplo real de chamada**

```bash
curl -X DELETE "https://pzaas.online/webhook/v1/logs-240285-239685" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: 4821"
```

> Operação idempotente: chamar de novo para um pedido já removido continua respondendo `200`.

### `GET /v1/logs`

Consulta os logs já registrados.

**Query params**

| Param | Obrigatório | Descrição |
|---|---|---|
| `pedido_id` | Não | Filtra apenas os logs daquele pedido. Se omitido, retorna a lista global |
| `limit` | Não | Máximo de itens retornados (padrão 50, máximo 200) |

**Resposta de sucesso — `200`**

```json
{
  "total": 2,
  "pedido_id": "4821",
  "logs": [
    {
      "timestamp": "2026-09-05T18:30:00.000Z",
      "received_at": "2026-09-05T18:30:00.120Z",
      "service": "forno",
      "level": "info",
      "message": "Pedido 4821 entrou no forno",
      "pedido_id": "4821",
      "meta": { "tempo_estimado_min": 12 }
    },
    {
      "timestamp": "2026-09-05T18:42:00.000Z",
      "received_at": "2026-09-05T18:42:00.080Z",
      "service": "fila-producao",
      "level": "info",
      "message": "Pedido 4821 liberado da fila",
      "pedido_id": "4821",
      "meta": null
    }
  ]
}
```

**Exemplo real de chamada**

```bash
curl "https://pzaas.online/webhook/v1/logs-240285-239685?pedido_id=4821&limit=10" \
  -H "x-api-key: turma2026"
```

### `GET /health`

Healthcheck obrigatório do contrato da turma.

**Resposta — `200`**

```json
{
  "status": "ok",
  "service": "logger-pzaas",
  "timestamp": "2026-09-05T18:45:00.000Z"
}
```

## Códigos de erro

| Código | Quando ocorre |
|---|---|
| `200` | `GET /v1/logs`, `PUT /v1/logs`, `PATCH /v1/logs`, `DELETE /v1/logs` ou `GET /health` bem-sucedidos |
| `201` | Log gravado com sucesso (`POST /v1/logs`) |
| `400` | Corpo inválido, campo obrigatório ausente, `level` fora do enum permitido, ou `x-pedido-id` ausente em `PUT`/`PATCH`/`DELETE` |
| `401` | `x-api-key` ausente ou inválida (não se aplica a `/health`) |
| `404` | `PATCH /v1/logs` chamado para um pedido sem nenhum log registrado |

Exemplo de erro `400`:

```json
{ "error": "bad_request", "message": "Campos obrigatórios ausentes: message" }
```

Exemplo de erro `401`:

```json
{ "error": "unauthorized", "message": "Header x-api-key ausente ou inválida" }
```

Exemplo de erro `404`:

```json
{ "error": "not_found", "message": "Nenhum log encontrado para este pedido" }
```

## Como os dados são guardados (Redis)

- `pzaas:logs:all` — lista com **todos** os logs recebidos (mais recente primeiro), em ordem de chegada
- `pzaas:logs:<pedido_id>` — lista só com os logs daquele pedido, quando o header `x-pedido-id` é enviado no `POST`
- Cada item da lista é uma string JSON (o `logEntry` serializado) — o próprio serviço faz o parse na hora de responder o `GET`

## Como consumir este serviço (para as outras duplas)

Sempre que o seu serviço quiser registrar algo, envie:

```
POST /v1/logs
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: <id do pedido, se já existir>

{ "service": "<nome do seu serviço>", "level": "info|warn|error|debug", "message": "<o que aconteceu>" }
```

De forma assíncrona sempre que possível (não bloqueie o fluxo do seu serviço esperando a resposta do Logger).

## Setup no n8n (para quem for rodar este workflow)

1. Importe `Logger-RA1-RA2.json` no n8n.
2. Em todos os nós **Redis** do fluxo (gravação, busca, substituição, patch e remoção), configure a credencial Redis:
   - Host: `redis` (ou o endpoint da conta Redis Cloud, se estiver usando a versão cloud)
   - Porta: `6379`
   - Senha: a senha Redis compartilhada pela turma (ver material de aula)
3. Renomeie o workflow para `Logger-<RA1>-<RA2>` (conforme instrução da planilha da turma).
4. Ative o workflow (toggle **Active**).
5. Confirme que `GET /health` responde `200` antes de integrar com as outras duplas.
6. Publique este link de documentação no comentário do seu grupo na planilha da turma.


