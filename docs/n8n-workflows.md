# Guia de evolução do workflow n8n

Este documento descreve, nó a nó, as 4 adições ao workflow existente no n8n
(`automacoes-n8n.m5jmff.easypanel.host`). A ordem das seções é a ordem
recomendada no canvas.

Fluxo final:

```
Webhook (form-lead-driva)
  -> [1] IF: segurança (origem + honeypot)
  -> Enriquecimento BrasilAPI (já existe)
  -> [3] OpenAI: resumo_ia + abordagem_sugerida
  -> Classificação em tier (já existe)
  -> Persistência (já existe: onde quer que os leads sejam salvos hoje)
       -> adicionar campos resumo_ia e abordagem_sugerida na gravação
  -> [1b] IF: tier != desqualificado
       -> [2] Twilio: alerta WhatsApp (somente se tier == quente)
       -> [4] HubSpot: upsert contato + deal
```

---

## 1. Segurança do webhook (origem + honeypot)

**Por quê:** hoje qualquer pessoa pode chamar o webhook diretamente (curl/Postman),
sem passar pelo formulário, e disparar enriquecimento + IA + CRM à toa. Isso gera
custo (chamadas de API, tokens de IA) e lixo no CRM.

**Onde:** logo depois do nó `Webhook`, antes de qualquer enriquecimento.

1. Adicione um nó **IF** logo após o `Webhook`.
2. Condição 1 (string): `{{$json.headers.origin}}` **is equal to**
   `https://kristhianno.github.io`
   - Isso funciona porque o navegador sempre envia o header `Origin` em
     requisições `fetch` cross-origin (o front-end está em `github.io`, o
     webhook está em `easypanel.host`).
3. Condição 2 (string, adicionar com "AND"): `{{$json.body.site}}` **is empty**
   - Esse é o campo honeypot que o formulário já envia oculto. Se vier
     preenchido, é bot.
4. Ramo **false** (qualquer condição falhou) → conectar direto a um nó
   **Respond to Webhook** retornando `200 OK` com corpo vazio (finja sucesso,
   não dê pista pro bot) e **não conectar nada depois** — a execução para aí.
5. Ramo **true** → segue para o restante do fluxo normalmente.

Reforço opcional (rápido de fazer, vale a pena): no nó `Webhook`, aba
**Options → Allowed Origins (CORS)**, preencha com
`https://kristhianno.github.io`. Isso bloqueia o **preflight** do navegador
para qualquer outro site, complementando a checagem acima (que cobre chamadas
diretas via curl/Postman, que não fazem preflight).

---

## 2. Alerta de WhatsApp para leads quentes

**Por quê:** fecha o ciclo captura → ação. O vendedor é avisado em segundos
quando um lead `quente` entra, em vez de depender de alguém abrir o Kanban.

**Onde:** depois do nó que já classifica o `tier` (o mesmo que hoje alimenta
`GET /leads-kanban`), em paralelo à gravação/HubSpot.

1. Adicione um nó **IF**: condição `{{$json.tier}}` **is equal to** `quente`.
2. No ramo **true**, adicione o nó nativo **Twilio** do n8n.
   - Credencial: crie uma credencial Twilio no n8n com `Account SID` e
     `Auth Token` (painel da Twilio → Console → Account Info).
   - Resource: `SMS` → funciona também para WhatsApp preenchendo os números
     no formato `whatsapp:+55...`.
   - **From**: número do sandbox/WhatsApp Business da Twilio, ex.
     `whatsapp:+14155238886` (sandbox) — em produção, migrar para um número
     verificado da Twilio ou WhatsApp Cloud API.
   - **To**: `whatsapp:+55<DDD><número do vendedor>` — pode vir de uma
     variável de ambiente do n8n (`{{$env.SALES_WHATSAPP_NUMBER}}`) para não
     ficar hardcoded no node.
   - **Body** (exemplo de template):
     ```
     🔥 Lead quente na Driva!
     Empresa: {{$json.razao_social || $json.empresa}}
     Contato: {{$json.nome}} — {{$json.telefone}}
     Ticket estimado: R$ {{$json.ticket_medio_mensal}}
     CNAE: {{$json.cnae_descricao}}
     ```
3. Teste primeiro no sandbox da Twilio (é grátis e não exige aprovação do
   Meta) antes de migrar para um número de produção.

---

## 3. Camada de qualificação com OpenAI

**Por quê:** gera um resumo e uma sugestão de abordagem por lead, no espírito
do "Driva Copilot" — mostra que a automação vai além de mover dado de A pra B.

**Onde:** logo depois do enriquecimento BrasilAPI (precisa de
`razao_social`, `cnae_descricao`, `situacao_cadastral` já disponíveis) e antes
da classificação de tier.

1. Adicione o nó nativo **OpenAI** (ou **Message a Model**, dependendo da
   versão do n8n).
   - Credencial: crie uma credencial OpenAI com sua API key.
   - Model: `gpt-4o-mini` (custo baixo, suficiente para essa tarefa).
   - Modo: **Chat** com "Response Format" = **JSON**.
2. **System prompt**:
   ```
   Você é um SDR da Driva, uma plataforma de inteligência de mercado B2B.
   Dado os dados de uma empresa que preencheu um formulário de contato,
   responda SOMENTE em JSON válido com exatamente estas chaves:
   {
     "resumo_ia": "resumo de 1-2 frases sobre a empresa e a oportunidade",
     "abordagem_sugerida": "1 frase com o melhor ângulo de abordagem comercial"
   }
   ```
3. **User prompt** (usar expressões do n8n para interpolar os dados do item
   atual):
   ```
   Empresa: {{$json.razao_social}}
   CNAE: {{$json.cnae_descricao}}
   Situação cadastral: {{$json.situacao_cadastral}}
   Ticket médio informado: R$ {{$json.ticket_medio_mensal}}/mês
   Nome do contato: {{$json.nome}}
   ```
4. Depois do nó OpenAI, adicione um nó **Set** (ou **Code**, se preferir
   `JSON.parse`) para extrair `resumo_ia` e `abordagem_sugerida` da resposta
   e mesclá-los de volta no item do lead, para que sigam junto no resto do
   fluxo (persistência, HubSpot, WhatsApp).

---

## 4. Integração com HubSpot

**Por quê:** a Driva já integra nativamente com HubSpot — em vez de um Kanban
paralelo, os leads também aparecem no CRM real, prontos pro time comercial
trabalhar onde já trabalha.

**Onde:** depois da classificação de tier, em paralelo ao alerta de
WhatsApp. Recomendo **não** enviar leads `desqualificado` pro CRM (adicionar
um IF antes: `{{$json.tier}}` **is not equal to** `desqualificado`).

1. Crie uma **Private App** no HubSpot (Configurações → Integrações →
   Private Apps) com escopos `crm.objects.contacts.write`,
   `crm.objects.contacts.read`, `crm.objects.deals.write`,
   `crm.schemas.contacts.write` (para criar propriedades customizadas).
2. Antes de rodar o workflow, crie as propriedades customizadas de contato no
   HubSpot (Configurações → Propriedades → Contato): `cnpj` (texto),
   `ticket_medio_mensal` (número), `lead_tier` (texto/enumeração),
   `resumo_ia` (texto multi-linha).
3. No n8n, adicione o nó nativo **HubSpot**:
   - Credencial: token da Private App criada no passo 1.
   - Resource: `Contact`, Operation: `Upsert` (usar `email` como chave de
     deduplicação).
   - Mapear: `firstname`/`lastname` (a partir de `nome`), `email`, `phone`
     (`telefone`), `company` (`razao_social` ou `empresa`), e as
     propriedades customizadas `cnpj`, `ticket_medio_mensal`, `lead_tier`
     (=`tier`), `resumo_ia`.
4. Adicione um segundo nó **HubSpot** (Resource: `Deal`, Operation:
   `Create`), associado ao contato criado acima, com o pipeline/estágio
   variando conforme `tier` (ex.: `quente` → estágio "Qualificado",
   `atencao` → estágio "Em nutrição").

---

## Variáveis de ambiente sugeridas (n8n → Settings → Environment)

| Variável | Uso |
|---|---|
| `SALES_WHATSAPP_NUMBER` | número do vendedor que recebe o alerta (item 2) |
| `ALLOWED_ORIGIN` | `https://kristhianno.github.io` (item 1, evita hardcode) |

Credenciais (Twilio, OpenAI, HubSpot) devem ficar sempre no **Credentials
Manager** do n8n, nunca em variáveis de ambiente ou hardcoded nos nós.
