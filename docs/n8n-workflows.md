# Guia de evolução do workflow n8n

O workflow completo e atualizado está em [`n8n/workflow.json`](../n8n/workflow.json)
— pronto para colar direto no n8n. Este documento cobre só os passos manuais
que não dá pra embutir num arquivo de workflow (credenciais, contas
externas, colunas de planilha).

## Como aplicar

1. Abra o workflow atual no n8n.
2. Selecione todo o canvas (Ctrl+A) e apague, **ou** crie um workflow novo —
   dependendo de quão confortável você está em substituir tudo de uma vez.
   (Alternativa mais segura: duplique o workflow atual antes, como backup.)
3. Abra `n8n/workflow.json`, copie todo o conteúdo (Ctrl+A, Ctrl+C).
4. Cole direto no canvas do n8n (Ctrl+V) — o n8n importa workflows colados
   como JSON automaticamente.
5. Siga o checklist abaixo antes de ativar/testar.

## O que mudou em relação ao fluxo original

```
Webhook - Formulário Site
  -> [NOVO] IF - Segurança (Origin + honeypot)
       false -> [NOVO] Respond - Bloqueado (200 vazio)
       true  -> Normalizar Payload (como antes)
  -> Enriquecer BrasilAPI / OpenCNPJ (sem mudança)
  -> Calcular Classificação (sem mudança na lógica de tier)
  -> [NOVO] Montar Prompt IA (monta o prompt da OpenAI)
  -> [NOVO] OpenAI - Qualificação (chamada HTTP para a API da OpenAI)
  -> [NOVO] Parse Resposta IA (extrai resumo_ia / abordagem_sugerida)
  -> Salvar Lead na Planilha (2 colunas novas: resumo_ia, abordagem_sugerida)
  -> IF Tier = Quente
       true  -> Montar Resumo (Quente) -> Enviar Resumo SDR (Quente)
                                        -> [NOVO] HubSpot - Upsert Contato
       false -> IF Tier = Atenção
                  true  -> Montar Resumo (Atenção) -> Enviar Resumo SDR (Atenção)
                                                     -> [NOVO] HubSpot - Upsert Contato
                  false -> Respond - Desqualificado
                        -> [NOVO] HubSpot - Upsert Contato
```

O alerta de WhatsApp via Evolution API (`Enviar Resumo SDR`) não mudou —
já funcionava e continua funcionando exatamente como antes, e só dispara
para os tiers `quente`/`atencao` (leads `desqualificado` não geram alerta,
só sincronizam com o HubSpot).

### Status do lead no board do HubSpot

Desde a última mudança, `Calcular Classificação` também calcula
`hs_lead_status` (propriedade nativa do HubSpot, a mesma que controla as
colunas do board de Contatos) a partir do tier:

| Tier | Coluna no HubSpot |
|---|---|
| `quente` | Oportunidade |
| `atencao` | Oportunidade |
| `desqualificado` | Nutrir |

Esse mapeamento está hardcoded no node `Calcular Classificação`
(constante `HUBSPOT_LEAD_STATUS`) usando os **rótulos** das opções como
valor. Isso só funciona de verdade se o valor interno de cada opção no
HubSpot for exatamente esse texto — **confirme antes de ativar** (ver
checklist item 5 abaixo). Se os valores internos forem diferentes (ex.
gerados automaticamente pelo HubSpot como `Oportunidade_2` ou algo em
inglês), ajuste a constante no código do node.

Antes desta mudança, leads `desqualificado` nunca chegavam ao HubSpot —
só ficavam na planilha. Agora todo lead que passa pelo formulário gera ou
atualiza um contato no HubSpot, cada um caindo na coluna correspondente
ao seu tier.

Optei por implementar a chamada da OpenAI e do HubSpot com o nó genérico
**HTTP Request** (o mesmo padrão já usado para BrasilAPI/OpenCNPJ), em vez
dos nodes nativos OpenAI/HubSpot do n8n — assim o JSON importa sem
depender de versões específicas desses nodes na sua instância.

---

## Checklist antes de ativar

### 1. Credencial da OpenAI

1. n8n → **Credentials → New → Header Auth**.
2. Name do header: `Authorization`. Value: `Bearer <sua-chave-nova>`
   (gere uma chave nova em https://platform.openai.com/api-keys — nunca
   reaproveite a que foi colada num chat).
3. Salve com um nome tipo `OpenAI Header Auth`.
4. Abra o nó **`OpenAI - Qualificação`** no workflow importado e selecione
   essa credencial no campo de autenticação (ela não vem pré-selecionada
   porque credenciais nunca são portáveis entre instâncias do n8n).

### 2. Credencial do HubSpot

1. Crie uma conta HubSpot (grátis) em https://app.hubspot.com/signup, se
   ainda não tiver.
2. **Configurações (⚙) → Integrações → Private Apps → Create a private
   app**.
   - Nome: `n8n - Lead Qualify`.
   - Scopes: `crm.objects.contacts.write`, `crm.objects.contacts.read`,
     `crm.schemas.contacts.write`.
   - Crie e copie o **token de acesso** (só aparece uma vez).
3. Crie as propriedades customizadas de contato **antes** de rodar o
   workflow: **Configurações → Propriedades → Contato → Criar
   propriedade**:
   - `cnpj` (texto de linha única)
   - `ticket_medio_mensal` (número ou texto — o node envia como string)
   - `lead_tier` (texto de linha única)
   - `resumo_ia` (texto multi-linha)
4. n8n → **Credentials → New → Header Auth** → Name: `Authorization`,
   Value: `Bearer <token do passo 2>`. Salve como `HubSpot Header Auth`.
5. Abra o nó **`HubSpot - Upsert Contato`** e selecione essa credencial.
6. **Confirme o valor interno das opções de `hs_lead_status`** antes de
   ativar: **Configurações → Propriedades → Propriedades de contato →
   Status do lead**. Cada opção do dropdown (Lead, Oportunidade, Nutrir,
   etc.) tem um rótulo (o que aparece na tela) e um valor interno (o que
   a API espera). O node `Calcular Classificação` está enviando
   `Oportunidade` e `Nutrir` como se fossem os valores internos — se o
   HubSpot gerou valores internos diferentes para essas opções, edite a
   constante `HUBSPOT_LEAD_STATUS` nesse node para usar os valores
   corretos. Envie um lead de teste de cada tier e confirme no board que
   ele caiu na coluna certa.

### 3. Colunas novas na planilha do Google Sheets

Na planilha `CadastroLeads` (aba `Leads`), adicione duas colunas no
cabeçalho, com esses nomes exatos (sensível a maiúsculas/minúsculas):
`resumo_ia` e `abordagem_sugerida`.

### 4. Teste

1. Ative o workflow.
2. Envie um lead de teste pelo formulário (https://kristhianno.github.io/lead_qualify).
3. Confira, na execução do n8n:
   - `IF - Segurança` aprovou (true).
   - `OpenAI - Qualificação` retornou 200 e `Parse Resposta IA` populou
     `resumo_ia`/`abordagem_sugerida`.
   - A linha na planilha tem as duas colunas novas preenchidas.
   - Se o ticket informado for ≥ R$ 5 mil/mês (`quente` ou `atenção`):
     o WhatsApp chegou (como já acontecia) **e** um contato novo apareceu
     no HubSpot.
4. Abra o Kanban (https://kristhianno.github.io/lead_qualify/kanban.html)
   e confirme que o card do lead mostra o bloco "Sugestão da IA".

---

## Notas de manutenção

- **Custo controlado**: `IF - Segurança` bloqueia qualquer chamada direta
  ao webhook antes de gastar com BrasilAPI/OpenAI/HubSpot.
- **Resiliência**: tanto `OpenAI - Qualificação` quanto `HubSpot - Upsert
  Contato` têm `onError: continueRegularOutput` — se a OpenAI ou o
  HubSpot falharem (rate limit, timeout), o lead ainda é salvo na
  planilha e o SDR ainda recebe o alerta de WhatsApp; só o
  enriquecimento por IA / sincronização de CRM daquele lead específico
  fica incompleto.
- **Sem chave real em arquivo**: o `.env`/`.env.example` na raiz do repo
  são só um checklist pessoal — as credenciais de verdade ficam sempre no
  Credentials Manager do n8n (Header Auth), nunca em texto no repositório.
