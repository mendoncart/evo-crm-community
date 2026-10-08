# Processor: Dify retorna 400/404 em todo chat

- **Tipo:** fix
- **Data:** 2026-10-07
- **Versão afetada:** `evoapicloud/evo-ai-processor-community` até 1.1.0
- **Imagem corrigida:** `ghcr.io/mendoncart/evo-ai-processor-community:1.1.0-rtm.1`
- **Arquivos alterados (submódulo `evo-ai-processor-community`):**
  - `src/services/providers/dify_service.py`
  - `src/services/adk/agents/external_agent.py`
  - `tests/unit/services/test_dify_service.py` (novo)

## Sintoma

O chat com um agente externo Dify devolve:

```
Error calling dify: Dify API error: 400
```

Nos logs da Dify não aparece nenhum uso da chave.

## Causa

Reproduzido direto na API da Dify, com o mesmo payload que o processor envia:

| Requisição | Resposta da Dify |
|---|---|
| `response_mode: blocking` num app **Agent** | `400 invalid_param: Agent Chat App does not support blocking mode` |
| `conversation_id` = ID da sessão do Evo | `404 not_found: Conversation Not Exists.` |

1. Com o tipo de bot **Chat Bot**, o processor usava `blocking`. Apps do tipo
   *Agent* na Dify só aceitam `streaming`.
2. O processor mandava o ID da sessão do Evo como `conversation_id`. A Dify só
   aceita IDs que ela mesma emitiu, então mesmo com o tipo **Agent** a chamada
   falhava com 404.

## Correção

- **Apps de chat (Chat Bot e Agent) usam sempre `streaming`.** O parser de SSE
  trata `message` (chatbot/chatflow) e `agent_message` (agent), e também
  repassa eventos `error`. O tipo *Text Generator* continua `blocking` em
  `/completion-messages`.
- **`conversation_id` emitido pela Dify:** na primeira mensagem o campo não é
  enviado. O ID que a Dify devolve vai para o estado da sessão do Evo
  (`dify_conversation_id`), via `state_delta` do evento de resposta, e é
  reutilizado nas mensagens seguintes. Se o ID guardado não existir mais na Dify,
  o processor abre uma conversa nova em vez de falhar.
- O erro exibido agora inclui a mensagem da Dify, não só o status HTTP.

Validado contra a API real da Dify (app *agent-chat*): com Chat Bot e com Agent,
o 1º turno responde, o 2º turno continua a mesma conversa (o bot lembra o
contexto) e um ID inválido gera conversa nova. Testes unitários: 7 novos, sem
regressão na suíte.

## Deploy

No compose:

```yaml
  evocrm_processor:
    image: ghcr.io/mendoncart/evo-ai-processor-community:1.1.0-rtm.1
```

Nada mais muda no serviço.
