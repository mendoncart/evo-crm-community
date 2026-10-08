# Gateway: 404 em /api/v1/integration-credentials

- **Tipo:** fix
- **Data:** 2026-10-07
- **Versão afetada:** `evoapicloud/evo-crm-gateway` até 1.1.0
- **Arquivos alterados:** `nginx/default.conf.template`, `docker-compose.swarm.yaml`
- **Relacionado:** [2026-10-07-gateway-agent-integrations-404.md](2026-10-07-gateway-agent-integrations-404.md)

## Sintoma

Ao abrir **Configurações → Credenciais de integração**
(`/settings/integration-credentials`), aparece o toast *"Falha ao carregar as
credenciais"*. No DevTools:

```
GET https://<API_DOMAIN>/api/v1/integration-credentials?page=1&pageSize=100 404 (Not Found)
```

Sem credenciais, não dá para configurar integrações que dependem delas
(por exemplo, a Dify).

## Causa

A rota `/integration-credentials` é servida pelo **core service** (Go, porta 5555),
em `evo-ai-core-service-community/pkg/integration_credential/handler/integration_credential_handler.go`.

No nginx do gateway, a regra que manda tráfego para o core só cobria
`agents|folders|mcp-servers|custom-mcp-servers|custom-tools`. Por isso,
`/api/v1/integration-credentials` caía no catch-all `^/api/v1/`, que aponta para
o **CRM** (Rails). O CRM não conhece essa rota e devolve 404.

`/api/v1/integrations/knowledge-nexus` também pertence ao core e tinha o mesmo
problema.

## Correção

Em `nginx/default.conf.template`, as duas rotas entraram na regra do core:

```nginx
location ~ ^/api/v1/(agents|folders|mcp-servers|custom-mcp-servers|custom-tools|integration-credentials|integrations/knowledge-nexus) {
    proxy_pass $evoai_service$request_uri;
```

A ordem não muda nada: essa regra vem antes de
`^/api/v1/integrations/[^/]+/callback` (processor), e nenhum callback começa com
`knowledge-nexus`.

## Workaround para imagens publicadas (sem rebuild)

Enquanto não sai uma imagem do gateway com a correção, o serviço do gateway no
`docker-compose.swarm.yaml` corrige o template na subida do container:

```yaml
entrypoint:
  - sh
  - -c
  - |
    sed -i 's#custom-mcp-servers|custom-tools)#custom-mcp-servers|custom-tools|integration-credentials|integrations/knowledge-nexus)#' /etc/nginx/templates/default.conf.template
    exec /docker-entrypoint.sh nginx -g 'daemon off;'
```

- Precisa ser `entrypoint`, e não `command`. O `/docker-entrypoint.sh` do nginx
  só renderiza os templates (envsubst dos `*_UPSTREAM`) quando o primeiro
  argumento é `nginx`.
- Numa imagem que já tem a correção, o padrão do `sed` não casa e nada muda.
  Dá para remover o bloco quando a imagem nova estiver em uso.
- O mesmo bloco funciona em stacks standalone (Portainer/Compose).

## Verificação

```bash
# o config gerado deve conter a rota
docker exec <container_gateway> grep integration-credentials /etc/nginx/conf.d/default.conf

# deve responder 200 (ou 401 sem token), não 404
curl -i https://<API_DOMAIN>/api/v1/integration-credentials -H "Authorization: Bearer <token>"
```

Depois, recarregue `/settings/integration-credentials`: o toast de erro não deve
mais aparecer.

## Fora do escopo (ruído no console)

- **CSP bloqueando `static.cloudflareinsights.com` e fontes:** quem injeta é o
  Cloudflare Web Analytics. É inofensivo; para sumir, desative o RUM no
  Cloudflare para o domínio do frontend.
- **Mensagem do i18next/Locize:** propaganda da biblioteca, não é erro.
