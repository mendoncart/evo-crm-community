# Registro de mudanças locais

Esta pasta documenta fixes, mudanças e novas features aplicadas neste fork,
em especial as que ainda não estão nas imagens publicadas no Docker Hub.

## Convenção

- Um arquivo por mudança: `AAAA-MM-DD-<slug-curto>.md`
- Tipo no topo: `fix`, `change` ou `feature`
- Cada registro diz: sintoma, causa, correção, workaround (se houver) e como verificar

## Índice

| Data       | Tipo | Registro                                                                                   |
|------------|------|--------------------------------------------------------------------------------------------|
| 2026-10-07 | fix  | [Gateway: 404 em /api/v1/integration-credentials](2026-10-07-gateway-integration-credentials-404.md) |
| 2026-10-07 | fix  | [Gateway: 404 em /api/v1/agents/:id/integrations/dify](2026-10-07-gateway-agent-integrations-404.md) |
| 2026-10-07 | fix  | [Frontend: agente externo não aceita edição e perde a credencial](2026-10-07-frontend-external-agent-config.md) |
| 2026-10-07 | fix  | [Processor: Dify retorna 400/404 em todo chat](2026-10-07-processor-dify-streaming-conversation.md) |
