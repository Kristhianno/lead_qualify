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

## Migração para prospecção ativa (busca por segmento/UF/cidade)

O formulário deixou de capturar um lead por vez (nome/e-mail/CNPJ/ticket)
e passou a **buscar empresas ativas por segmento + estado + cidade**,
mostrar os resultados na tela e rodar cada empresa encontrada pelo mesmo
pipeline de qualificação (IA, tier, WhatsApp, HubSpot). Novo webhook:
`POST /webhook/prospect-search-ALPHADATA` (o antigo
`/webhook/form-lead-ALPHADATA` foi desativado).

### Fonte de dados: Minha Receita

[Minha Receita](https://minhareceita.org) é uma API pública, gratuita e
open-source (github.com/cuducos/minha-receita) sobre os dados abertos de
CNPJ da Receita Federal. A instância pública só aceita `cnae` + `uf` como
filtro no servidor (sem filtro de cidade ou situação cadastral, por
limitação de custo de hospedagem do próprio projeto) — por isso o filtro
de cidade + `situacao_cadastral == ATIVA` é feito depois, dentro do n8n
(node `Filtrar Cidade e Situacao Ativa`), junto com um cap de 30 empresas
por busca. A resposta de busca já vem com o mesmo schema completo do
lookup individual (e-mail, telefone, `qsa`/sócios) — BrasilAPI/OpenCNPJ só
entram como fallback quando uma empresa específica vem sem `qsa`.

Antes de mexer no workflow, valide isso ao vivo:

```bash
curl -s "https://minhareceita.org/?cnae=6201501&uf=SP&limit=5" | jq .
```

Campos confirmados na resposta (2026-08): `municipio` (string maiúscula
sem acento, ex. `"SAO PAULO"`), `descricao_situacao_cadastral` (ex.
`"ATIVA"`), `porte` (`"MICRO EMPRESA"` / `"EMPRESA DE PEQUENO PORTE"` /
`"DEMAIS"`), `qsa[].nome_socio`, `ddd_telefone_1`, `email`, `capital_social`
— e o parâmetro `cnae` espera dígitos crus (`6201501`), não formatado.

### Nodes novos

`Webhook - Buscar Empresas` → `IF - Segurança` (reaproveitado) →
`Normalizar Busca` → `Mapear Segmento para CNAE` →
`Buscar Empresas - Minha Receita` → `Extrair Lista de Empresas` →
`Filtrar Cidade e Situacao Ativa` → `IF - Dados Completos da Minha
Receita` → (`Adaptar Dados Minha Receita` ou fallback
`Enriquecer BrasilAPI`/`Enriquecer OpenCNPJ`, sem mudança de lógica) →
`Calcula e Classifica tier` → ... (pipeline existente) → `HubSpot - Upsert
Contato` → `Agregar Resultado Final` (novo) → `Respond - Resultado da
Busca` (novo).

### Nodes removidos

`Webhook - Formulário Site1`, `Normalizar Payload`, `Respond - Quente`,
`Respond - Atenção`, `Respond - Desqualificado` — substituídos por um único
`Respond - Resultado da Busca`, alimentado por `Agregar Resultado Final`.

### A pegadinha do `items[0]`/`.first()` em execução com múltiplos itens

Os Code nodes do pipeline original (`Formatar Dados BrasilAPI/OpenCNPJ`,
`Calcula e Classifica tier`, `Montar Prompt IA`, `Parse Resposta IA`,
`Montar Resumo (Quente/Atenção)`) foram escritos como "Run Once for All
Items" usando `items[0].json` e `$('Normalizar Payload').first().json` —
seguro só porque sempre passava exatamente 1 lead por execução. Buscar até
30 empresas na mesma execução faria esses nodes processarem só a empresa
#1 (ou colar os dados da empresa #1 em todas as outras), silenciosamente.
Todos foram convertidos para **"Run Once for Each Item"**
(`"mode": "runOnceForEachItem"` nos parâmetros do node), o código passou a
usar `$json` no lugar de `items[0].json`, `$('<node>').item.json` no lugar
de `.first().json` (o acessor `.item` resolve o item específico que gerou
o item atual, ao contrário de `.first()`), e o `return` passou a ser um
objeto único `{ json: {...} }`, não mais um array — é a diferença de
contrato entre os dois modos do Code node do n8n.

### `HubSpot - Upsert Contato`

`idProperty` trocado de `'email'` para `'cnpj'` (empresa prospectada nem
sempre tem e-mail, e não há "pessoa" que preencheu o formulário — quem
existe é a empresa). Isso exige marcar a propriedade `cnpj` como **valor
único** no HubSpot (checkbox na criação da propriedade) — sem isso o
upsert em lote falha. `firstname`/`ticket_medio_mensal` saíram do body;
entraram `segmento`, `porte`, `capital_social`.

### Tier por porte + capital social

Empresa prospectada não informa `ticket_medio_mensal` (ninguém preencheu
formulário). O node `Calcula e Classifica tier` passou a classificar por
`porte` + `capital_social` (dados que a própria Receita já devolve):

```js
const CAPITAL_SOCIAL_QUENTE = 100000;
const CAPITAL_SOCIAL_ATENCAO = 20000;
// porte === 'DEMAIS' && capital_social >= CAPITAL_SOCIAL_QUENTE   -> quente
// porte === 'DEMAIS' (capital menor) OU
// porte === 'EPP' && capital_social >= CAPITAL_SOCIAL_ATENCAO     -> atencao
// resto (ME, ou EPP com capital baixo)                            -> desqualificado
```

São valores de partida, não calibrados — **recalibrar depois de rodar
buscas reais** e ver a distribuição de porte/capital do segmento
pesquisado (mesmo espírito de constante tunável que `HUBSPOT_LIFECYCLE_STAGE`
acima).

### Mapeamento segmento → CNAE

Definido no node `Mapear Segmento para CNAE`, também editável/ilustrativo:

| Segmento (slug) | CNAE |
|---|---|
| `tecnologia` | 6201501 |
| `varejo_vestuario` | 4781400 |
| `alimentacao` | 5611201 |
| `construcao_civil` | 4120400 |
| `saude_odonto` | 8630504 |
| `educacao_idiomas` | 8593700 |
| `beleza_estetica` | 9602501 |
| `contabilidade_consultoria` | 6920601 |
| `logistica_transporte` | 4930202 |
| `ecommerce` | 4791100 |

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
   - `cnpj` (texto de linha única) — marque **"Valor deve ser único para
     este objeto"** ao criar; o upsert em lote usa `cnpj` como
     `idProperty`, então isso é obrigatório, não opcional.
   - `segmento` (texto de linha única)
   - `porte` (texto de linha única)
   - `capital_social` (número ou texto — o node envia como string)
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

Na planilha `CadastroLeads` (aba `Leads`), confira/adicione no cabeçalho,
com esses nomes exatos (sensível a maiúsculas/minúsculas):
`resumo_ia`, `abordagem_sugerida` (da migração anterior) e, novas desta
migração, `segmento`, `cidade`, `uf`, `socios`. Aproveite para conferir se
o cabeçalho já usa `cnae_descricao`/`fonte_enriquecimento` (com essa
grafia) — o node `Salvar Lead na Planilha` sempre produziu esses nomes,
mas o cache de schema do editor tinha `cnae_decricao`/`fonte_enriquecedora`
(grafia diferente); se o cabeçalho real da planilha seguiu o cache errado
em vez do que o código sempre gerou, essas duas colunas nunca receberam
dado — corrija o cabeçalho enquanto já está mexendo na planilha.

### 4. Teste

1. Ative o workflow.
2. Rode o `curl` de validação da Minha Receita (seção "Migração para
   prospecção ativa" acima) antes de testar pela UI.
3. Faça uma busca de teste pelo formulário
   (https://kristhianno.github.io/lead_qualify) — segmento + estado + uma
   cidade menor, para limitar o volume no primeiro teste (ex.:
   Curitiba/PR).
4. Confira, na execução do n8n:
   - `IF - Segurança` aprovou (true).
   - `Filtrar Cidade e Situacao Ativa` produziu ≤30 itens, todos com
     `descricao_situacao_cadastral == 'ATIVA'` e município batendo a
     cidade buscada.
   - Cada empresa passou individualmente por `OpenAI - Qualificação` (não
     só a primeira — é a regressão que o fix `runOnceForEachItem` existe
     para evitar).
   - A planilha ganhou uma linha por empresa, com `segmento`/`cidade`/
     `uf`/`porte`/`capital_social`/`socios` preenchidos.
   - Empresas `quente`/`atenção`: o WhatsApp chegou, uma mensagem por
     empresa (não uma mensagem combinada, não zero).
   - HubSpot mostra um contato upsertado por empresa (todos os tiers), com
     `cnpj`/`segmento`/`porte`/`capital_social`/`lifecyclestage` corretos.
5. No navegador, confirme que `index.html` mostrou a lista de resultados
   (não travou em "Buscando...").
6. Abra o Kanban (https://kristhianno.github.io/lead_qualify/kanban.html)
   e confirme que os cards novos aparecem na coluna certa, com o bloco
   "Sugestão da IA".
7. Teste também o caso de zero resultados (segmento+estado+cidade
   improvável) e confirme que `index.html` mostra "nenhuma empresa
   encontrada" em vez de travar ou dar erro.

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

**Nota pós-migração para prospecção:** o bloco de "maior/menor ticket
médio" do e-mail depende de `ticket_medio_mensal`, campo que só as linhas
antigas (captura manual de lead) têm — linhas novas de prospecção vêm com
esse campo vazio. Esse fluxo não foi alterado nesta migração (continua
lendo `ticket_medio_mensal` normalmente), então o relatório tende a esvaziar
esse bloco conforme a planilha for enchendo majoritariamente com resultados
de busca. Ajustar isso é um passo futuro, fora do escopo desta migração.

---

## Notas de manutenção

- **Custo controlado**: `IF - Segurança` bloqueia qualquer chamada direta
  ao webhook antes de gastar com Minha Receita/BrasilAPI/OpenAI/HubSpot; o
  cap de 30 empresas por busca (`Filtrar Cidade e Situacao Ativa`) evita
  que uma busca de segmento+UF muito genérica dispare dezenas de chamadas
  de OpenAI/HubSpot de uma vez.
- **Limitação de filtro da Minha Receita**: a instância pública só filtra
  por `cnae`+`uf` no servidor — cidade e situação cadastral são filtradas
  depois, no n8n, sobre o lote de até 1000 resultados trazido por
  `Buscar Empresas - Minha Receita`. Buscas de segmento+UF muito amplos
  combinados com cidades pequenas podem retornar poucos ou nenhum
  resultado após o filtro; isso é esperado, não é bug.
- **Resiliência**: tanto `OpenAI - Qualificação` quanto `HubSpot - Upsert
  Contato` têm `onError: continueRegularOutput` — se a OpenAI ou o
  HubSpot falharem (rate limit, timeout), a empresa ainda é salva na
  planilha e o SDR ainda recebe o alerta de WhatsApp (quando aplicável);
  só o enriquecimento por IA / sincronização de CRM daquela empresa
  específica fica incompleto.
- **Sem chave real em arquivo**: o `.env`/`.env.example` na raiz do repo
  são só um checklist pessoal — as credenciais de verdade ficam sempre no
  Credentials Manager do n8n (Header Auth), nunca em texto no repositório.
  A Minha Receita não exige chave/autenticação.
