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

`Calcular Classificação` calcula um valor de propriedade do HubSpot a
partir do tier, para o contato cair na coluna certa do **board de
Contatos**:

| Tier | Coluna no board |
|---|---|
| `quente` | Lead |
| `atencao` | Oportunidade |
| `desqualificado` | Nutrir |

**Atenção — pegadinha já vivida neste projeto:** o board de Contatos
desta conta é agrupado pela propriedade **`lifecyclestage`** (Lifecycle
Stage), **não** por `hs_lead_status`. As duas são propriedades nativas
do HubSpot que existem em qualquer conta, então é fácil confundir uma
com a outra — o node já chegou a gravar em `hs_lead_status` (que
realmente existe e aceita valores, só que não é a propriedade que esse
board usa), e o sintoma foi contatos sendo criados normalmente mas
sumindo do board (contagem 0 em todas as colunas). Confirmado via
`GET https://api.hubapi.com/crm/v3/properties/contacts/lifecyclestage`
com o token do Private App — **se o board não bater com os valores
abaixo, rode esse GET de novo antes de mexer em qualquer coisa**, pois
os valores internos das opções são específicos desta conta.

Os valores internos confirmados nesta conta (constante
`HUBSPOT_LIFECYCLE_STAGE` no node `Calcular Classificação`):

| Opção (rótulo) | Valor interno | Tier que grava aqui |
|---|---|---|
| Lead | `lead` | `quente` |
| Opportunity (exibido como "Oportunidade") | `opportunity` | `atencao` |
| Nutrir (opção customizada) | `1398914058` | `desqualificado` |

`lead`/`opportunity` são os valores padrão do HubSpot (não mudam entre
contas). `1398914058` é um id numérico **auto-gerado** no momento em que
a opção customizada "Nutrir" foi criada nesta conta — não é a string
`"nutrir"` nem nada previsível, e não vai se repetir se essa opção for
recriada em outra conta/instância. Se algum dia o board parar de bater
de novo, o primeiro suspeito é essa constante ter ficado desatualizada
em relação ao valor real da opção.

Antes da correção, leads `desqualificado` nunca chegavam ao HubSpot — só
ficavam na planilha. Depois, todo lead que passa pelo formulário gera ou
atualiza um contato no HubSpot, mas o campo gravado ainda era
`hs_lead_status`. A correção trocou o campo para `lifecyclestage`, o que
foi validado criando manualmente um contato de teste
(`lead.teste.automatiza@example.com`, tier `quente`,
`lifecyclestage: lead`) direto via API e confirmando que ele aparece na
coluna "Lead" do board.

Optei por implementar a chamada da OpenAI e do HubSpot com o nó genérico
**HTTP Request** (o mesmo padrão já usado para BrasilAPI/OpenCNPJ), em vez
dos nodes nativos OpenAI/HubSpot do n8n — assim o JSON importa sem
depender de versões específicas desses nodes na sua instância.

## Novo fluxo independente: Relatório diário de leads por e-mail

```
Schedule Trigger - Relatório Diário (todo dia às 08:00) ─┐
Manual Trigger - Relatório Diário (sob demanda)          ─┴─> Buscar Todos Leads (Relatório)
  -> Calcular Estatísticas do Relatório (conta por tier; acha maior/menor ticket_medio_mensal)
  -> Montar Email Relatório (HTML) (monta assunto + corpo HTML)
  -> Gmail - Enviar Relatório Diário (envia para ekriator@gmail.com)
```

Esse fluxo é totalmente independente dos outros dois (captura de lead e
listagem do Kanban) — tem seus próprios gatilhos e sua própria leitura da
planilha (`Buscar Todos Leads (Relatório)`), em vez de reaproveitar
`Buscar Todos Leads`/`Agrupar Leads em Array` do fluxo do Kanban. Isso é
proposital: se o relatório dependesse dos nodes do Kanban, toda visita ao
board público (`kanban.html`) dispararia um e-mail de relatório como
efeito colateral.

O relatório é **cumulativo** (conta todas as linhas que já existem na
planilha até o momento da execução), não "leads desde o último
relatório" — não há filtro de janela de tempo.

O tier `desqualificado` corresponde à coluna "Nutrir" do board do HubSpot
(ver tabela de `lifecyclestage` acima) — por isso aparece como 🌱 Nutrir
no e-mail, embora internamente o campo continue se chamando
`desqualificado` na planilha/código.

Se a planilha estiver vazia, o node de leitura tem `alwaysOutputData`
ativado para garantir que o fluxo não pare em silêncio — o e-mail ainda
sai, com o texto "Nenhum lead cadastrado na planilha ainda."

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
6. **Confirme o valor interno das opções de `lifecyclestage`** antes de
   ativar — é a propriedade que controla o board de Contatos, não
   `hs_lead_status` (as duas existem em qualquer conta HubSpot, mas só
   uma controla esse board; ver "Status do lead no board do HubSpot"
   acima para o histórico de como isso já causou contatos sumindo do
   board). O jeito mais confiável de confirmar não é pela UI, é via API:
   ```
   GET https://api.hubapi.com/crm/v3/properties/contacts/lifecyclestage
   Authorization: Bearer <token do Private App>
   ```
   O array `options` da resposta traz `label` (o que aparece na tela) e
   `value` (o que a API espera de verdade). Se os valores não baterem com
   a tabela em "Status do lead no board do HubSpot", edite a constante
   `HUBSPOT_LIFECYCLE_STAGE` no node `Calcular Classificação`. Envie um
   lead de teste de cada tier e confirme no board que ele caiu na coluna
   certa.

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

### 5. Credencial do Gmail (relatório diário)

1. n8n → **Credentials → New → Gmail account** (OAuth2).
2. Siga o fluxo OAuth do Google e autorize com a conta que deve aparecer
   como remetente (pode ser a mesma conta dona do Google Sheets ou outra
   conta Gmail, desde que tenha a Gmail API habilitada no Google Cloud
   Console do projeto usado para o OAuth client).
3. Salve a credencial com um nome tipo `Gmail account`.
4. Abra o nó **`Gmail - Enviar Relatório Diário`** no workflow importado
   e selecione essa credencial (assim como OpenAI/HubSpot, ela não vem
   pré-selecionada porque credenciais não são portáveis entre instâncias
   do n8n).
5. Diferente de OpenAI/HubSpot (implementados com HTTP Request genérico
   de propósito, para não depender de versão de node específica), este
   nó usa o node nativo do Gmail. Se o n8n mostrar um aviso de versão do
   node após colar o JSON, abra o nó e ajuste para a versão instalada na
   sua instância.

### 6. Teste do relatório diário

1. No canvas, clique em **Execute workflow** a partir do node **`Manual
   Trigger - Relatório Diário`** para rodar o fluxo sob demanda, sem
   esperar até as 08:00.
2. Confira na execução:
   - `Buscar Todos Leads (Relatório)` leu todas as linhas da aba `Leads`.
   - `Calcular Estatísticas do Relatório` bateu com os totais por tier
     que aparecem no Kanban, e identificou corretamente o maior/menor
     `ticket_medio_mensal`.
   - O e-mail chegou em ekriator@gmail.com com o HTML formatado e os
     blocos de maior/menor ticket preenchidos (nome, empresa, porte,
     CNAE, resumo da IA).
3. Se possível, teste também o caso de planilha vazia (ex. numa cópia de
   teste sem linhas de dado) e confirme que o e-mail sai com o texto
   "Nenhum lead cadastrado" em vez de travar a execução.
4. Ative o workflow (`active: true`) para que o Schedule Trigger dispare
   às 08:00 todo dia — lembre que isso ativa o workflow inteiro,
   incluindo os outros dois fluxos já existentes.

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
