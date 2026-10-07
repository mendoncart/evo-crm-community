# Frontend: configuração de agente externo (Dify etc.) não aceita edição e perde a credencial

- **Tipo:** fix
- **Data:** 2026-10-07
- **Versão afetada:** `evoapicloud/evo-ai-frontend-community` até 1.1.0
- **Arquivos alterados:**
  - `evo-ai-frontend-community/src/components/agents/configuration/ModelApiPanel.tsx`
  - `evo-ai-frontend-community/src/components/agents/AgentWizardModal.tsx`
- **Relacionado:** [2026-10-07-gateway-agent-integrations-404.md](2026-10-07-gateway-agent-integrations-404.md)

## Sintoma

1. Em `/agents/<id>/edit` → **Configuração**, os campos do provider externo
   (Credencial do cofre, URL da API, API Key) parecem habilitados, mas não
   aceitam digitação nem seleção. O formulário marca "URL da API é obrigatória".
2. Um agente criado pelo wizard com uma credencial do cofre aparece com
   **"Nenhuma (usar valor embutido)"**.
3. Ao conversar com o agente:
   ```
   500: {'error': 'Error building External agent: Dify apiUrl is required', ...}
   ```

## Causa

**Edição (`ModelApiPanel.tsx`).** O painel montava o `data` do
`ExternalAgentConfig` só com o `provider`, a cada render, e o `onChange`
repassava ao pai apenas o `provider`. Qualquer valor digitado ou selecionado
era descartado no render seguinte, então o input parecia "travado". Como o
formulário ficava sempre vazio, também não dava para salvar a integração pela
tela de edição.

**Criação (`AgentWizardModal.tsx`).** O wizard gravava `apiUrl`/`apiKey`/`botType`,
mas **não** gravava o `credential_id`. Com a chave inline aposentada pelo cofre,
a integração ficava sem credencial. Além disso, um `apiKey` vazio era enviado e
podia sobrescrever o fallback guardado.

O processor (`src/services/providers/dify_service.py`) exige `apiUrl` e `apiKey`
na configuração resolvida. Por isso, uma integração salva incompleta gera o 500.

## Correção

- `ModelApiPanel.tsx`: o painel guarda o estado completo do formulário
  (`externalFormData`) e só propaga para o pai a troca de `provider`. O estado é
  reiniciado quando o `provider` muda. Os campos do provider continuam sendo
  persistidos pelo botão **Salvar Configuração** do próprio formulário.
- `AgentWizardModal.tsx`: envia o `credential_id` (exceto para Typebot, que não
  usa credencial) e omite o `apiKey` quando vazio, como o formulário de edição
  já fazia.

## Como recuperar um agente já afetado

Depois de subir o frontend corrigido (e o gateway com os dois fixes de rota):

1. Abra o agente → **Configuração**.
2. Selecione a **Credencial do cofre**, preencha a **URL da API** (por exemplo,
   `https://api.dify.ai/v1`) e o **Tipo de Bot**.
3. Clique em **Salvar Configuração**.

Teste o chat de novo.

## Sem rebuild?

Não tem. O frontend é um bundle compilado, então é preciso buildar a imagem
`evo-ai-frontend-community` a partir deste repositório. Enquanto isso, a
alternativa é apagar o agente e recriá-lo pelo wizard **digitando a API Key
inline** (sem escolher credencial do cofre), desde que o campo esteja liberado.
Se o campo estiver bloqueado pelo cofre, só o build resolve.

## Verificação

- Em `/agents/<id>/edit`, digitar nos campos deve funcionar, e a credencial
  selecionada deve continuar selecionada.
- Depois de salvar, `GET /api/v1/agents/<id>/integrations/dify` deve trazer
  `config.apiUrl` e `config.credential_id`.
- O chat não deve mais retornar `apiUrl is required`.
