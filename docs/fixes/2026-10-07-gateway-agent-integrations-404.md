# Gateway: 404 em /api/v1/agents/:id/integrations/dify (agentes externos)

- **Tipo:** fix
- **Data:** 2026-10-07
- **Versão afetada:** `evoapicloud/evo-crm-gateway` até 1.1.0 (as que incluem o commit `df3aa71`)
- **Arquivos alterados:** `nginx/default.conf.template`, `docker-compose.swarm.yaml`
- **Relacionado:** [2026-10-07-gateway-integration-credentials-404.md](2026-10-07-gateway-integration-credentials-404.md)

## Sintoma

Ao editar um agente externo (Dify, Flowise, n8n, OpenAI, Typebot) em
`/agents/<id>/edit`, o console mostra:

```
GET https://<API_DOMAIN>/api/v1/agents/<id>/integrations/dify 404 (Not Found)
Error loading external integration: AxiosError: Request failed with status code 404
Error loading integration: AxiosError: Request failed with status code 404
```

A configuração já salva (URL da API, tipo do bot, credencial escolhida) não é
carregada no formulário. Remover a integração (`DELETE`) também dá 404.

## Causa

O commit `df3aa71` mandou **todo** `^/api/v1/agents/[^/]+/integrations/` para o
**processor**, para que as integrações OAuth (Google Calendar, GitHub etc.)
funcionassem.

Só que o processor implementa apenas os providers OAuth: asana, atlassian,
canva, github, google-calendar, google-sheets, hubspot, linear, monday, notion,
paypal e supabase (veja `evo-ai-processor-community/src/api/*_routes.py`).

Os providers de agente externo são gravados pelo **core**
(`GET/DELETE /agents/:id/integrations/:provider` em
`evo-ai-core-service-community/pkg/agent_integration/handler/agent_integration_handler.go`).
Como iam para o processor, que não tem essas rotas, o resultado era 404.

## Correção

Em `nginx/default.conf.template`, a regra do processor agora casa só os
providers OAuth. Os demais caem na regra do core, que vem logo abaixo:

```nginx
location ~ ^/api/v1/agents/[^/]+/integrations/(asana|atlassian|canva|github|google-calendar|google-sheets|hubspot|linear|monday|notion|paypal|supabase)(/|$) {
    proxy_pass $processor_service$request_uri;
```

Se um provider OAuth novo for adicionado ao processor, ele precisa entrar
nessa lista.

## Workaround para imagens publicadas (sem rebuild)

O `entrypoint` do gateway em `docker-compose.swarm.yaml` traz um segundo `sed`:

```sh
sed -i 's#/integrations/ {#/integrations/(asana|...|supabase)(/|$$) {#' $$T
```

No compose, o `$$` é o escape de `$`. Numa imagem já corrigida o padrão não
casa e nada muda.

## Não é bug: campo de API key bloqueado

Com o cofre de credenciais ativo (`GET /api/v1/integration-credentials/migration-state`
retorna `retired.external_agents: true`, o normal numa instalação nova), o campo
**API Key** do Dify fica desabilitado de propósito: a chave inline foi
aposentada. A credencial deve ser escolhida no seletor **de credencial** que
fica acima dos campos do provider. Os campos **URL da API** e **Tipo de bot**
continuam editáveis.

Se a rota `migration-state` não funcionar (fix anterior não aplicado), o guard
falha e a chave inline continua editável.

## Verificação

```bash
docker exec <container_gateway> grep -n "integrations/(asana" /etc/nginx/conf.d/default.conf

# agente sem integração salva: o core responde (404 JSON do core ou 200), não o 404 do FastAPI
curl -i https://<API_DOMAIN>/api/v1/agents/<id>/integrations/dify -H "Authorization: Bearer <token>"
```

Para distinguir os dois 404: o do processor (FastAPI) tem corpo `{"detail":"Not Found"}`.
O do core segue o envelope de erro do core.
