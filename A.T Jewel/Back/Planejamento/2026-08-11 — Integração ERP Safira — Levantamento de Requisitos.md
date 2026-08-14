# Integração ERP Safira × CRM — Levantamento de Requisitos

**Reunião:** [CRM A.T. JEWEL] STB <> CONEXA — Alinhamento Sobre Dados de Estoque
**Participantes:** Alessandro Naime Pontes (Conexa / ERP Safira), Yerlon Magalhães (STB), Lucas Barbosa (STB)
**Ausentes:** Caio Medeiros, Thiago
**Analisado em:** 11/08/2026 — *data da reunião a confirmar*

Levantamento feito a partir da transcrição, confrontado linha a linha com o schema
e as rotas do `at-jewel-back` no estado de 11/08/2026.

---

## 1. Sumário executivo

A reunião definiu o modelo de dados de estoque da integração. Confrontando com o
código, **a maior parte do que foi acordado não existe no CRM** — e um dos itens
não é adição, é substituição de um modelo já em uso.

| #   | Achado                                                                                                                                                                                                                  | Gravidade |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| 1   | **Estoque hoje é um inteiro no produto** (`produtos.estoque_atual`). O acordado é uma tabela dimensionada por empresa/local. Não é campo novo — é troca de modelo, com 7 pontos de leitura dependendo do formato atual. | **Alta**  |
| 2   | **Não existe nenhuma noção de empresa no backend.** Zero ocorrências de "empresa" em todo o `src`. Estoque e vendas precisam dela.                                                                                      | **Alta**  |
| 3   | **Já existe uma tabela `consignacoes`** (migração 24), com semântica *diferente* da acordada. Dois modelos concorrentes para o mesmo fato.                                                                              | **Alta**  |
| 4   | **Cliente perdeu CPF/CNPJ e endereço de propósito** (migração 03, decisão de LGPD documentada). A reunião pede de volta. É decisão de privacidade, não técnica.                                                         | **Alta**  |
| 5   | **O ERP não tem telefone de vendedora** — Alessandro conferiu os 20 cadastros: só código e nome. O WhatsApp da Elena nunca virá da integração.                                                                          | Média     |
| 6   | Forma de pagamento é `ENUM` no CRM; o ERP tem cadastro com ID próprio.                                                                                                                                                  | Média     |
| 7   | Não existe cadastro de fornecedores — só a string livre `referencia_fornecedor`.                                                                                                                                        | Média     |
| 8   | Não existe rota de ingestão de clientes pelo ERP (`POST /erp/clientes`).                                                                                                                                                | Média     |

**A pergunta mais importante ficou em aberto e não foi levantada na reunião:**
o ERP vai enviar **saldo** (snapshot da posição) ou **movimento** (lançamentos)?
Toda a modelagem de estoque muda conforme a resposta. Ver §6.

---

## 2. O modelo acordado

Tabela única de estoque, validada por todos ao final:

```
empresa | grupo de local | local | produto | quantidade
```

Todos os campos são string, exceto a quantidade. Exemplo enviado pelo Alessandro:

```
001 | ESTOQUE CLIENTE | 001 - Lucas | AN001 | 1
001 | ESTOQUE         | 001         | AN001 | 10
```

**Semântica de cada campo:**

- **empresa** — o grupo da A.T. Jewel opera N empresas no mesmo sistema. Não são
  necessariamente filiais: uma trabalha joias, outra trabalha outro segmento, e o
  cadastro de produto é compartilhado. O mesmo anel pode existir na empresa 1 e na 5.
- **grupo de local** — *classificação* do estoque, não quantidade. Distingue estoque
  próprio de consignação para cliente e de consignação de fornecedor. Sem ele, o
  `local` "cliente 001" seria indistinguível do armário "001".
- **local** — dentro do grupo. No grupo próprio é o armário/setor (a Fabrícia já criou
  "at-wear", "home" e o "disponível" que já existia). No grupo consignação é o
  cliente ou o fornecedor.
- **quantidade** — **pode ser negativa**, e o sinal carrega regra de negócio:

| Grupo de local            | Sinal permitido | Significado                             |
| ------------------------- | --------------- | --------------------------------------- |
| Estoque próprio           | ≥ 0             | peças em posse e disponíveis            |
| Consignação para cliente  | ≥ 0             | peças emprestadas, ainda da loja        |
| Consignação de fornecedor | ≤ 0             | peças que a loja **deve** ao fornecedor |

A consignação de fornecedor gera **dois lançamentos**: positivo no estoque disponível
(a peça está lá e pode ser vendida) e negativo no grupo do fornecedor (a peça não é
dela). Vendida a peça, o disponível zera e o negativo permanece — vira a lista de
compra obrigatória.

**Distinção que o Alessandro fez questão de firmar:** transferência entre empresas do
grupo **não é consignação**. Consignação é sempre com terceiro externo — cliente ou
fornecedor.

---

## 3. Matriz — o que já tem × o que falta

| Entidade                  | Hoje no CRM                           | Acordado                   | Situação             |
| ------------------------- | ------------------------------------- | -------------------------- | -------------------- |
| Produto — cadastro        | Tabela completa, CRUD por API key     | mantém                     | ✅ pronto             |
| Produto — estoque         | `estoque_atual INT` (escalar)         | tabela dimensionada        | ⚠️ **substituir**    |
| Produto — identificadores | `id` UUID + `codigo_erp` VARCHAR      | UUID + ID numérico do ERP  | ⚠️ ambíguo           |
| Produto — classificações  | 7 strings soltas                      | strings (ou FK, opcional)  | ✅ ok / dívida        |
| Empresa                   | **nada**                              | cadastro + FK              | ❌ criar              |
| Estoque                   | **nada** (só o escalar)               | tabela nova                | ❌ criar              |
| Grupo de local / local    | **nada**                              | cadastros de apoio         | ❌ criar              |
| Consignação cliente       | `consignacoes` [SYS], origem CRM      | vem do ERP, via estoque    | ⚠️ **conflito**      |
| Consignação fornecedor    | **nada**                              | saldo negativo no estoque  | ❌ criar              |
| Venda — cabeçalho         | Tabela completa + ingestão            | mantém                     | ✅ pronto             |
| Venda — empresa           | **nada**                              | obrigatório                | ❌ criar              |
| Venda — local de origem   | **nada**                              | prever (opcional)          | ❌ criar              |
| Forma de pagamento        | `ENUM` de 8 valores                   | cadastro com ID            | ⚠️ converter         |
| Cliente — cadastro        | Tabela, sem PII fiscal                | + CPF/CNPJ, IE, endereço   | ⚠️ **conflito LGPD** |
| Cliente — observação      | `observacao_geral` (cifrado)          | idem                       | ✅ pronto             |
| Cliente — ingestão ERP    | **nada** (`/erp/clientes` não existe) | necessário                 | ❌ criar              |
| Fornecedor                | string `referencia_fornecedor`        | cadastro completo          | ❌ criar              |
| Vendedora — cadastro      | Tabela completa                       | ERP só manda código + nome | ⚠️ ver RF-INT-11     |
| Vendedora — ingestão ERP  | **nada**                              | necessário                 | ❌ criar              |
| Moeda                     | implícito BRL                         | BRL apenas                 | ✅ fora de escopo     |
| Custo médio por empresa   | `valor_custo` único                   | pendente de validação      | ⏸️ aguardando        |

---

## 4. Requisitos

### RF-INT-01 — Cadastro de empresas

**Hoje:** não existe. `grep -ri "empresa" src --include=*.ts` retorna **zero linhas**.

**Acordado:** N empresas do mesmo grupo, no mesmo sistema, compartilhando o cadastro
de produtos. O Alessandro envia a tabela para que o relacionamento seja por ID —
Yerlon comentou em reunião que prefere o ID ao nome.

**Trabalho:** tabela `empresas` (`id` UUID, `codigo_erp` UNIQUE, `nome`, `ativo`),
ingestão, e FK a partir de estoque e vendas.

**Pré-requisito de:** RF-INT-02 e RF-INT-06. É a primeira coisa a existir.

---

### RF-INT-02 — Tabela de estoque *(substitui o modelo atual)*

**Hoje:** a migração 16 adicionou `produtos.estoque_atual INT NOT NULL DEFAULT 0` e
`data_entrada_estoque`. Um número por produto, sem dimensão de empresa ou local.

**Quem escreve hoje:** apenas `POST /produtos` e `PATCH /produtos/:id`
(`produtos.controller.ts:162`), autenticados por API key com escopo `produtos:write`.
**O webhook `POST /erp/produtos` não tem o campo** — `ErpProdutoDto` não declara
`estoque_atual`. Ou seja: quem sincronizar por lá deixa o estoque zerado, e não há
erro nenhum indicando isso.

**Quem lê hoje** — e portanto quebra se a coluna sumir:

| Consumidor                 | Arquivo                           | Uso                                     |
| -------------------------- | --------------------------------- | --------------------------------------- |
| Inventário (KPI)           | `analytics.repository.ts:344,355` | total de peças e valor total do estoque |
| Giro por família           | `analytics.repository.ts:269-284` | soma por família                        |
| Alertas de estoque         | `produto.repository.ts:110`       | `WHERE estoque_atual <= $1`             |
| Produtos disponíveis       | `produto.repository.ts:123`       | `WHERE estoque_atual > 0`               |
| Elena — estoque baixo      | `agentes-data.repository.ts:65`   | `WHERE estoque_atual <= 2`              |
| Elena — disponíveis        | `agentes-data.repository.ts:83`   | filtro de sugestão                      |
| Elena — análise de produto | `analisar-produto.use-case.ts:38` | texto do prompt                         |

**Acordado:** tabela `estoque` com empresa, grupo de local, local, produto, quantidade.

**Recomendação:** criar a tabela nova **e manter `produtos.estoque_atual` como coluna
derivada** — soma do grupo "estoque próprio" em todas as empresas, recalculada na
ingestão. Os sete consumidores acima continuam funcionando sem reescrita, e as telas
migram para a visão dimensionada uma a uma. Trocar tudo de uma vez transforma uma
integração em uma refatoração de analytics.

**Risco de não fazer:** os KPIs de inventário e giro passam a ler de uma fonte que
ninguém alimenta e reportam zero silenciosamente — sem erro, sem log, sem alerta.

---

### RF-INT-03 — Grupo de local e local

**Acordado:** `grupo de local` classifica o tipo de estoque; `local` identifica o
ponto dentro do grupo.

**Valores esperados** (a confirmar com o Alessandro, que envia as tabelas):
`ESTOQUE` (próprio), `CONSIGNACAO_CLIENTE`, `CONSIGNACAO_FORNECEDOR`.

**Regra de sinal** — vale como `CHECK` no banco, não só como validação de aplicação:
grupo de fornecedor aceita apenas quantidade ≤ 0; estoque próprio nunca negativo.
Alessandro foi explícito: *"no estoque disponível não pode estar negativo"* e *"no
grupo de local fornecedor só vai existir peça negativa ou zerada"*.

**Nota:** a Fabrícia **já cadastrou** os locais no ERP ("at-wear", "home", e o
"disponível" preexistente), mas o Alessandro ainda vai validar se ela está separando
estoque de verdade ou apenas agrupando produtos. O modelo suporta os dois — não é
motivo para esperar.

---

### RF-INT-04 — Consignação: conflito com a tabela existente

**Hoje:** a migração 24 criou `consignacoes`, marcada `[SYS]` — *"dado operacional do
sistema novo, não vem do ERP Safira"*. Um admin registra pela tela; destino é
`CLIENTE` ou `VENDEDORA`; status `ABERTA` / `DEVOLVIDA` / `VENDIDA`. Tem CRUD
(`GET`, `POST`, `PATCH`) e permissões próprias (`consignacoes:read/write`).

**Acordado na reunião:** consignação vem do ERP, aparece como linha do estoque, e é
**sempre externa** — Alessandro: *"a consignação é sempre para cliente dela que não é
empresa. É sempre externo"*.

**Três divergências concretas:**

1. **Origem.** Uma nasce no CRM, a outra chega do ERP. Se as duas coexistirem sem
   regra, os números divergem e ninguém sabe qual está certo.
2. **`destino_tipo = 'VENDEDORA'`** não tem equivalente no ERP. A tabela do CRM
   permite consignar para vendedora; o modelo do Alessandro, não.
3. **Consignação de fornecedor não existe** em lugar nenhum do CRM — nem tabela, nem
   coluna, nem conceito.

**Decisão necessária** (nossa, não do cliente):

- **(a)** `consignacoes` vira visão derivada do estoque — uma fonte só, telas mantidas;
- **(b)** as duas coexistem com escopos declarados: CRM para prova rápida na loja, ERP
  para consignação formal — exige rotular a origem em cada registro;
- **(c)** `consignacoes` é descontinuada e a tela passa a ler do estoque.

Recomendo **(a)**. É a única que não deixa dois números competindo.

**Contexto que pesa:** Alessandro confirmou ao final que há consignação em uso —
*"tem consignação, então a gente vai precisar separar"* e *"entrei aqui e tem monte de
consignação"*. Não é hipótese; é volume real esperando.

---

### RF-INT-05 — Identificadores de produto *(ação atribuída ao Lucas)*

**Hoje:** `produtos.id` UUID (nosso) + `produtos.codigo_erp VARCHAR(50) UNIQUE`.
O ERP já grava o nosso UUID do lado dele — Alessandro: *"os produtos que eu mando
para você, eu pego o ID que você gerou e gravo no meu produto"*.

**O que a reunião revelou:** o ERP tem **dois** identificadores por entidade — o ID
numérico interno (ex.: `472324` no vendedor) e o "código", uma espécie de SKU reduzido
(ex.: `017`, `AN001`). Hoje `codigo_erp` guarda **um dos dois, e não está registrado
qual**.

**Ação acordada:** o CRM passa a guardar os dois — o SKU visível e o ID numérico interno.

**⚠️ Confirmar antes de mexer.** `codigo_erp` é a chave de idempotência da
sincronização de produtos e o alvo de `itens_venda.produto_codigo_erp`. Se o ERP
passar a mandar o ID numérico onde antes mandava o SKU, **a sincronização duplica o
catálogo inteiro** — o `UNIQUE` não impede, porque são valores diferentes.

**Pergunta ao Alessandro:** o `codigo_erp` que chega hoje em `POST /erp/produtos` é o
ID numérico ou o código reduzido?

---

### RF-INT-06 — Empresa e local de origem na venda

**Hoje:** `vendas` não tem empresa. `ErpVendaDto` também não.

**Acordado:** empresa é **obrigatória** na venda. Local/estoque de origem: o Alessandro
não vê necessidade hoje, mas sugeriu prever o campo — *"se quiser colocar também, não
é ruim"*.

**Trabalho:** coluna + FK + campo no DTO. Aditivo e barato, desde que RF-INT-01 exista.

---

### RF-INT-07 — Formas de pagamento: ENUM → cadastro

**Hoje:** `CREATE TYPE forma_pagamento AS ENUM ('dinheiro', 'pix', 'cartao_credito',
'cartao_debito', 'transferencia', 'crediario', 'cheque', 'outro')`, usado em
`pagamentos_venda` e validado no DTO via `IsIn(FORMAS_PAGAMENTO)`.

**Acordado:** o ERP tem cadastro próprio, com ID e classificação (cartão, boleto,
dinheiro, PIX...). Yerlon assumiu criar o cadastro no CRM.

**Detalhe de mapeamento:** o Alessandro trata PIX, TED e DOC como a **mesma coisa** —
*"no frigir dos ovos vai cair do mesmo jeito"*. O ENUM atual separa `pix` de
`transferencia`. O mapeamento precisa ser explícito, senão a distribuição por forma de
pagamento muda de leitura sem ninguém perceber.

**Recomendação:** tabela `formas_pagamento` (`id`, `codigo_erp`, `nome`, `classificacao`)
com FK em `pagamentos_venda`, **mantendo o ENUM como coluna de classificação**. Assim
`/analytics/distribuicao-pagamento` continua funcionando enquanto a granularidade nova
entra.

---

### RF-INT-08 — Cadastro de fornecedores

**Hoje:** não existe. Só `produtos.referencia_fornecedor VARCHAR(100)` — texto livre,
descrito na migração 01 como servindo *"sem precisar do cadastro do fornecedor"*.

**Consequência já ativa:** `/analytics/giro-estoque` agrupa por essa string
(`giroEstoquePorFornecedor`). Variação de grafia — "Antica" e "Ântica" — vira dois
fornecedores distintos no relatório.

**Acordado:** cadastro com a mesma estrutura de cliente, mais observação. Campos que o
Alessandro confirmou existirem e serem usados: razão social, nome fantasia, CPF/CNPJ
(mesmo campo, com flag PF/PJ), inscrição estadual, endereço completo, telefone.
Preenchimento é irregular — *"tem alguns casos que ela alimenta, sim"*.

**Pré-requisito de:** RF-INT-04 (consignação de fornecedor precisa de a quem atribuir
o saldo negativo).

---

### RF-INT-09 — Cliente: campos fiscais e endereço ⚠️ decisão de privacidade

**Hoje:** `clientes` tem nome, nome fantasia, tipo de pessoa, telefones, e-mail,
observações e limite de crédito. **CPF, RG, data de nascimento, gênero, endereço
completo e inscrição estadual foram removidos deliberadamente.** A migração 03
documenta a decisão:

> *"Campos REMOVIDOS em relação ao ERP (data minimization, LGPD). Justificativa:
> nenhum desses campos é necessário para o atendimento personalizado que é a finalidade
> do sistema."*

**Acordado na reunião:** cliente com a mesma estrutura do fornecedor — o que reintroduz
CPF/CNPJ, inscrição estadual e endereço.

**Isto não é uma mudança técnica.** É reverter uma decisão de LGPD tomada e registrada.
Antes de implementar, é preciso: declarar a finalidade de cada campo reintroduzido,
cifrar o que for PII (o padrão AES-256-GCM já existe) e registrar a mudança de decisão.

**Observação já existe** — `observacao_geral`, cifrada. O item da reunião "adicionar
observação em clientes" está atendido no CRM; falta apenas no fornecedor, que não existe.

**Nota operacional do Alessandro:** o CPF frequentemente não é preenchido — *"é
segmento que o pessoal não gosta muito de se identificar"*. **Não serve como chave.**
A chave continua sendo `codigo_erp`.

---

### RF-INT-10 — Ingestão de clientes pelo ERP

**Hoje:** **não existe `POST /erp/clientes`.** O `ErpController` expõe apenas
`POST /erp/produtos` e `POST /erp/vendas`, ambos sob `SafiraAuthGuard` (`x-safira-key`).

O que existe é `POST /clientes`, autenticado por API key com escopo `clientes:write` —
desenhado para os agentes de IA criarem cliente a partir do WhatsApp, **sem `evento_id`
e portanto sem idempotência**.

**Decisão necessária:** ou se cria `POST /erp/clientes` com `evento_id` no padrão dos
outros dois webhooks, ou se assume que o integrador usa a API key e aceita o risco de
reprocessamento duplicar cliente.

**Relacionado — dois caminhos concorrentes já existem para produto:**
`POST /erp/produtos` (`x-safira-key`, com `evento_id`, **sem** estoque) e
`POST /produtos` (`x-api-key`, escopo `produtos:write`, **com** `estoque_atual`).
Fazem quase a mesma coisa com campos diferentes. Vale definir qual é o caminho oficial
antes de replicar a ambiguidade em clientes, vendas e estoque.

---

### RF-INT-11 — Vendedoras: o ERP não tem o telefone

**Levantado na reunião:** o cadastro de vendedor no ERP tem endereço, e-mail e telefone
disponíveis, **mas a Fabrícia não preenche nenhum**. Alessandro conferiu os 20
registros ao vivo: *"Nenhum. Olhei todos. Tudo zerado. Só código e nome."* E explicou
que é típico do segmento — *"o pessoal põe o menos informação possível"*.

**Hoje no CRM:** `vendedoras` tem `email` e `whatsapp_interno` cifrados, com hash HMAC
para lookup. Não existe `POST /erp/vendedoras`; só `POST /vendedoras` por JWT. E **não
existe tela de vendedoras no painel**.

**Duas consequências diretas:**

1. **O WhatsApp da vendedora nunca virá da integração.** Isso deixa de ser hipótese e
   vira fato confirmado pelo dono do ERP. A tela `/admin/vendedoras` sai de
   "conveniência" para **único caminho possível** de preencher o campo — confirmando o
   pré-requisito bloqueante já registrado no plano da Elena.
2. **Risco de atribuição de venda.** `vendas.vendedora_codigo_erp` resolve contra
   `vendedoras.codigo_erp`. Se uma venda chegar antes da vendedora existir no CRM, a FK
   fica NULL e a atribuição se perde **sem erro**. Como o ERP vai começar a mandar
   vendas, isso é uma janela real.

**Trabalho:** ingestão de vendedoras (código + nome, com upsert **parcial** que preserve
`whatsapp_interno` e `especialidades` — dados que só existem no CRM), e a tela de edição
no painel.

---

### RF-INT-12 — Classificações de produto *(dívida aceita)*

**Hoje:** sete strings soltas em `produtos` — categoria, família, coleção, cor, tamanho,
tipo de pedra, coleção de pedra. `/produtos/facetas` deriva os filtros daí.

**Acordado:** o ERP tem uma tabela única de classificações (tipo + valor), com IDs.
Alessandro ofereceu o relacionamento e argumentou a favor: *"se ela mudar, vocês vão ter
que alterar as tabelas"* — mudança de nomenclatura no ERP se propagaria sozinha.

**Consenso da reunião:** manter string por ora. *"Pode deixar do jeito que tá."*

**Registrado como dívida:** reclassificação no ERP exige alteração de dados do nosso
lado, e a facetagem herda erros de grafia.

---

### RF-INT-13 — Moeda *(fora de escopo, registrado)*

O ERP guarda valor + moeda e tem tabela de cotação para conversão. **100% do catálogo
da A.T. Jewel está em Real** — Alessandro verificou ao vivo. O CRM assume BRL
implicitamente em todos os `DECIMAL(15,2)`.

Sem trabalho agora. Se a operação passar a usar outra moeda, o impacto é amplo
(produtos, vendas, itens, pagamentos, todo o analytics) e não é adição de coluna.

---

### RF-INT-14 — Custo médio por empresa *(aguardando cliente)*

**Hoje:** `produtos.valor_custo`, único. A migração 01 já registra a ambiguidade:
*"⚠️ Pendente: distinguir custo cadastrado vs. custo médio vs. último custo (ver US02)."*

**Situação:** o ERP suporta custo médio individualizado por empresa; a A.T. Jewel usa
custo médio único. Alessandro vai validar com a Fabrícia.

**Se mudar:** o custo passa a ser atributo da linha de estoque, não do produto — e a
margem por venda precisa saber de qual empresa saiu a peça.

---

### RF-INT-15 — Transferência entre empresas

Existe no ERP e **não é consignação**. Se o envio for de saldo consolidado por
(empresa, local, produto), a transferência aparece sozinha como mudança nos dois saldos
e não precisa de evento próprio.

Se o envio for de movimento, precisa de tipo de lançamento. Ver §6.

---

## 5. Impacto no front

| Item | Impacto |
|---|---|
| `/admin/vendedoras` | **Bloqueante.** Único caminho para o WhatsApp da vendedora (RF-INT-11). Já estava no plano da Elena; agora com justificativa confirmada pelo ERP. |
| Telas de produto | Estoque deixa de ser um número e vira posição por empresa/local. Exige seletor de empresa. |
| Tela de consignações | Depende da decisão do RF-INT-04. Se `consignacoes` virar visão derivada, a tela muda de origem. |
| `src/lib/auth/scopes.ts` | Escopos novos a espelhar conforme as rotas nascerem (`estoque:*`, `fornecedores:*`, `empresas:*`). O arquivo já ficou dessincronizado uma vez — ver `Front/Atualizações/2026-08-11.md`. |
| KPIs de inventário | Se `estoque_atual` deixar de ser mantida, dashboards reportam zero sem erro visível. |

---

## 6. Perguntas em aberto para o Alessandro

Ordenadas por quanto travam o desenho.

1. **Saldo ou movimento?** A tabela acordada tem "quantidade", o que sugere *snapshot
   da posição*. Mas a consignação de fornecedor com saldo negativo tem cara de
   *lançamento*. São modelos diferentes: snapshot é idempotente e descarta histórico;
   movimento exige razão e reconstrói posição. **Nada foi decidido — e define tudo.**
2. **Gatilho e frequência do envio.** Evento por mudança, como em produtos? Estoque
   muda a cada venda — o volume é outra ordem de grandeza. Ou lote periódico?
3. **`codigo_erp` de produto hoje é o ID numérico ou o código reduzido?** (RF-INT-05 —
   risco de duplicar o catálogo).
4. **Quais entidades ganham webhook `/erp/*`?** Empresas, locais, clientes,
   fornecedores, vendedores, formas de pagamento, estoque — todos com `evento_id`?
5. **Consignação de fornecedor entra no mesmo endpoint de estoque** ou terá o seu?
6. **Autenticação:** `x-safira-key` (webhook Safira) ou `x-api-key` com escopo? Hoje
   produto aceita os dois caminhos, com campos diferentes (RF-INT-10).
7. **Quem debita o estoque na venda?** Se o ERP é fonte da verdade e envia posição, o
   CRM não deve decrementar sozinho — hoje não decrementa, o que por acaso está certo.
   Convém firmar antes que alguém "corrija".

---

## 7. Sequenciamento sugerido

| Fase | Entrega | Depende de |
|---|---|---|
| **0** | Respostas de §6, principalmente a #1 | Alessandro |
| **1** | `empresas` + `estoque` + ingestão, com `estoque_atual` mantido como derivado | Fase 0 |
| **2** | Empresa na venda | Fase 1 |
| **3** | Decisão do RF-INT-04 e unificação da consignação | Fase 1 |
| **4** | Fornecedores + consignação de fornecedor | Fase 3 |
| **5** | Cadastro de formas de pagamento | independente |
| **6** | Ingestão de vendedoras + tela `/admin/vendedoras` | independente — **destrava a Elena** |
| **7** | Campos fiscais do cliente | decisão de LGPD (RF-INT-09) |

As fases 5 e 6 não dependem da resposta de §6 e podem começar já. A fase 6 tem o maior
retorno imediato: fecha o bloqueio da Elena e reduz o risco de perda de atribuição de
venda descrito no RF-INT-11.

---

## 8. Ações registradas na reunião

| Responsável | Ação | Requisito |
|---|---|---|
| Alessandro | Validar com a Fabrícia a necessidade real de separação de estoque | RF-INT-03 |
| Alessandro | Verificar o uso e a configuração de consignação no sistema | RF-INT-04 |
| Alessandro | Validar necessidade de custo médio por empresa | RF-INT-14 |
| Alessandro | Revisar a estrutura de dados enviada pelo Yerlon | RF-INT-02 |
| **Lucas** | **Adicionar identificador numérico do ERP ao cadastro de produto** | **RF-INT-05** |
| Yerlon | Adicionar observação a clientes e fornecedores | RF-INT-08 / RF-INT-09 |
| Yerlon | Criar cadastro de formas de pagamento | RF-INT-07 |

**Nota sobre a ação do Lucas:** não executar antes da pergunta #3 de §6 ser respondida.
Sem saber qual identificador chega hoje, adicionar a coluna nova cria ambiguidade em vez
de resolvê-la.

---

## 9. Observações

- Alessandro relatou possível procedimento na vista na semana seguinte à reunião.
  Dúvidas de integração devem ir pelo grupo, sem depender de agendar reunião — ele
  ofereceu explicitamente esse canal.
- A transcrição está gravada e foi autorizada por todos os participantes.
- Referência de código: `at-jewel-back` em 11/08/2026, migrações 01 a 25.

---

## Documentos relacionados

- [[Aplicar migração de banco]] — como cada migração deste levantamento vai ser
  escrita e aplicada
- [[2026-08-12]] — o controle de migrações, pré-requisito das migrações 26+, e
  os achados sobre o ambiente de produção
