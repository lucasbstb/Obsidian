# 2026-09-03 — Documento de Melhorias v1.0 — Batimento

Chegou hoje o **Documento de Melhorias v1.0**, do Thiago Parente, com 26 itens
(MEL-01 a MEL-26) tirados de uma reunião de alinhamento. Ele foi escrito a partir
da transcrição, sem consultar o código — então boa parte do que pede já existe, e
três itens pedem o **oposto** de decisões tomadas de propósito.

Isto é o batimento item a item, verificado arquivo por arquivo. Nenhuma linha de
código mudou.

|                             |        |
| --------------------------- | ------ |
| contemplado totalmente      | **2**  |
| parcialmente                | **15** |
| não existe                  | **7**  |
| conflita ou está indefinido | **2**  |

Legenda das tabelas: **T** total · **P** parcial · **N** não existe · **C** conflito.

> [!info] As tabelas abaixo são a fotografia de **03/09**. O que mudou desde
> então está na seção seguinte — as tabelas ficam como estavam para o batimento
> continuar legível como diagnóstico.

---

# O que mudou desde este batimento

## 04/09 — dezenove dos vinte e seis fechados

Produtos, Catálogo e Clientes saíram do papel, mais Ocorrências e as mensagens
das agentes. Quinze itens viraram código; quatro fecharam por decisão do Lucas,
sem escrever uma linha.

|        | Item                             | 03/09 | agora | como                                                                   |
| ------ | -------------------------------- | ----- | ----- | ---------------------------------------------------------------------- |
| MEL-01 | ID do produto no cadastro        | N     | **T** | modal de Nova peça, com o botão antes do Exportar                      |
| MEL-02 | Miniatura e coluna de ID         | N     | **T** | miniatura com proxy no back, e coluna de Código com clique para copiar |
| MEL-03 | Perfis Estoquista e Marketing    | P     | **T** | decisão do Lucas: **não há essa segregação**                           |
| MEL-04 | Juros manuais sobre preço        | P     | **T** | a regra mudou (ver abaixo) e parcelas/juro ficaram editáveis no card   |
| MEL-05 | Textos em tooltip                | N     | **T** | componente `Dica`; cinco parágrafos viraram ⓘ                          |
| MEL-06 | Aprovação pelo marketing         | P/C   | **T** | decisão do Lucas: o fluxo atual **já é** o que se queria               |
| MEL-07 | Observação individual por imagem | P     | **T** | coluna na referência, lista vertical, e a nota viaja na exportação     |
| MEL-08 | Cabeçalho padronizado / capa     | P     | **T** | capa escolhida entre as referências, aparecendo no card da lista       |
| MEL-09 | Exportação em alta qualidade     | P     | **T** | decisão: **já é o melhor que temos** (ver abaixo)                      |
| MEL-10 | PDF como referência              | N     | **T** | aceita PDF até 100 MB, com cartão de arquivo no lugar da miniatura     |
| MEL-11 | Destaque do botão Anexar         | N     | **T** | sólido e dourado, no lugar do tracejado apagado                        |
| MEL-12 | "ChatGpt" → "Agente"             | N     | **T** | e as duas frases que **mentiam** sobre o mecanismo foram corrigidas    |
| MEL-15 | Regra de encerramento             | P     | —     | **Movido** para a frente do WhatsApp: o desfecho vem de LER a conversa, não de digitar à mão. Trava na mesma pergunta do MEL-14 |
| MEL-18 | Papéis de Helena e Anastácia      | P/C   | **T** | **O documento é que está errado.** São cinco personas, não três — ver abaixo |
| MEL-20 | Módulo de Ocorrências             | P     | **T** | cliente (opcional), fotos por episódio, e filtro por cliente que faz a consulta virar histórico |
| MEL-21 | Lista de clientes anonimizada     | P     | **T** | a tabela de clientes, com telefone e o nome da vendedora. O recorte por carteira é o que "anonimiza" |
| MEL-23 | Proteção de dados de terceiros    | P     | **T** | `clientes:read_all` + `EscopoClientesService`, espelhando o de vendas  |
| MEL-24 | Mensagens com opções em lista     | C     | **T** | **deixou de ser conflito** — o Lucas decidiu que lista vale no canal interno |
| MEL-25 | API de vendas para o Alessandro   | P     | **T** | **feito e entregue** — informado pelo Lucas em 04/09 |
| MEL-26 | Limpeza da tela de Vendas         | ?     | **T** | os 1.269 registros de seed apagados em 03/09 |

O relato do dia, com os porquês e os erros: [[2026-09-04]].

### As cinco personas (MEL-18)

O documento fala em "Helena e Anastácia" e troca os papéis. No sistema são
**Elena** e **Anastasia**, e são **cinco prompts**, não dois:

| persona | com quem fala | editável na tela de Prompts |
|---|---|---|
| Anastasia — Analytics | quem abre o painel | sim |
| Anastasia — Triagem | **cliente novo**, por WhatsApp | sim |
| Elena — Catálogo / Estoque | quem abre o painel | sim |
| **Elena interna** | **a vendedora**, por WhatsApp | **não — só no código** |
| **Anastasia gestão** | **a administração**, por WhatsApp | **não — só no código** |

O resumo: **Anastasia fala com quem é de fora e com a gestão; Elena fala com
quem é de dentro.**

As duas últimas são importadas como constante nos use cases e nunca passam pelo
registro que alimenta a tela. **Quem ajustar a "Elena" ali muda a do painel** — a
que conversa com a vendedora continua igual até o próximo deploy. O Lucas optou
em 04/09 por deixar assim.

### O MEL-24 deixou de ser conflito

O batimento classificou como **C** porque as personas proíbem lista e markdown de
propósito. Em 04/09 o Lucas decidiu o contrário: **pedir a lista de vendedoras, de
produtos, de qualquer coisa listável deve vir como lista** — para admin,
vendedora e estoquista.

Mudado nas duas personas do canal interno, com quatro restrições que existem para
a lista funcionar no WhatsApp: numeração em vez de hífen (marcador de markdown
renderiza mal), uma linha de abertura, teto de dez itens dizendo quantos ficaram
de fora, e um item por linha.

**A triagem ficou de fora**: ali a conversa é com cliente novo, e é justamente
onde lista numerada vira robô.

### Três coisas que este batimento errou

**O MEL-01 não era "maior do que o documento imagina".** Eu escrevi que criar
cadastro no painel seria decidir quem é a fonte da verdade. O conflito já estava
resolvido no código: o `upsertByCodigoErp` casa por `codigo_erp`, então peça
cadastrada aqui é **atualizada** quando o Safira mandar a mesma, não duplicada.

**O MEL-06 não era conflito.** Li o pedido como "trocar quem aprova"; ele é "o
marketing precisa ver", e isso já existia.

**O MEL-09 já estava pronto** no escopo que interessa. Medi as imagens no bucket:
a exportação leva o PNG tratado sem recomprimir, e o original do WhatsApp — que
tem mais pixels — é JPEG de 141 KB contra 1,9 MB. Mais pixels, menos qualidade. O
teto de 1024 é do `gpt-image-1`, não da exportação.

### A regra de preço mudou

Decisão do Lucas em 04/09, e é a de maior alcance:

```
juro escrito na legenda  ->  à vista × (1 + juros/100)
nada escrito             ->  à vista, e ponto
```

Substitui a regra da casa — dividir por 0,80 em 10X e 0,90 em 6X, 25% embutidos.
**Peça já cadastrada com juro NULL passa a ser impressa mais barata:**
R$35.920,00 em 10X saía 10 X R$4.490,00 e passa a sair 10 X R$3.592,00. Sem
migração de dados — não há dado errado, mudou a conta na hora de exibir.

### Um defeito de infraestrutura achado no caminho

O `experimental.proxyClientMaxBodySize` do Next tem padrão de **10 MB**, e ele
corta o corpo **sem devolver erro**. Todo upload grande do painel estava quebrado
— inclusive o **envio do catálogo final montado fora**, que está no ar com limite
de 100 MB no back e nunca tinha sido testado com arquivo de gráfica.

Corrigido em `next.config.ts`. **Exige reiniciar o container do front em
produção**, não basta `git pull`.

---

# O batimento de 03/09

## Produtos

|        | Item                               |       | O que existe hoje                                                                                                                                     |
| ------ | ---------------------------------- | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEL-01 | ID do produto no cadastro          | **N** | Não existe tela de cadastro de produto. `/admin/produtos` é listagem read-only; o `codigoErp` só aparece na exportação e na busca livre               |
| MEL-02 | Miniatura e coluna de ID na tabela | **N** | Nenhuma das duas. Mas `foto_url` já é coluna do produto e o `GET /produtos` já devolve — o tipo do front é que a omite                                |
| MEL-03 | Perfis Estoquista e Marketing      | **P** | Os dois existem desde a migração 21. MARKETING tem três permissões; ESTOQUISTA também tem `catalogo:write`, então a segregação pedida não é a de hoje |
| MEL-04 | Juros manuais sobre preço          | **P** | A coluna `juros_percentual` e o cálculo existem. A **única** entrada é a legenda do WhatsApp — não há tela nem endpoint                               |

O MEL-01 é maior do que o documento imagina: o produto **nasce no ERP Safira**, não
no painel. Criar cadastro aqui é decidir quem é a fonte da verdade, e isso não é
ajuste de tela.

## Catálogo

|        | Item                                     |         | O que existe hoje                                                                                                                                    |
| ------ | ---------------------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEL-05 | Textos em tooltip, linguagem formal      | **N**   | Não existe componente de tooltip; são oito parágrafos explicativos em tela                                                                           |
| MEL-06 | Aprovação de fotos pelo marketing        | **P/C** | O ciclo existe inteiro — seis status, só `APROVADA` entra no catálogo, o "X" tira e é reversível. Mas quem aprova é **quem fotografou, no WhatsApp** |
| MEL-07 | Várias imagens com observação individual | **P**   | Várias imagens sim; observação por arquivo não — observações são linhas soltas, sem vínculo com o arquivo                                            |
| MEL-08 | Cabeçalho padronizado                    | **P**   | Nome, ID, contagem, criador e data existem. **Capa por catálogo não**: reaproveita a primeira referência, e no card da lista nunca mostra imagem     |
| MEL-09 | Exportação em alta qualidade             | **P**   | ZIP com as fotos sem recompressão e o CSV do descritivo. Mas a imagem nasce **1024×1024**                                                            |
| MEL-10 | Upload de PDF como referência            | **N**   | Referência aceita só JPEG, PNG e WebP. PDF é aceito noutro lugar — no catálogo montado que volta do marketing                                        |
| MEL-11 | Destaque do botão Anexar                 | **N**   | Hoje é o elemento **menos** destacado do cartão: borda tracejada, cor apagada                                                                        |
| MEL-12 | Trocar "ChatGpt" por "Agente"            | **N**   | Duas ocorrências visíveis, as duas no front. O back nunca nomeia o provedor                                                                          |

## Atendimento e acompanhamento

|        | Item                           |         | O que existe hoje                                                                                                                                                       |
| ------ | ------------------------------ | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEL-13 | Aba única, eliminando Demandas | **P/C** | Parte de um engano — ver adiante                                                                                                                                        |
| MEL-14 | Timeline visual por vendedora  | **P**   | Já existe em `/admin/auditoria`: cartão por vendedora e linha do tempo com hora, etapa, cliente, relato e próximo contato. Plota **episódios e relatos**, não conversas |
| MEL-15 | Regra de encerramento          | **P**   | `desfecho` aceita VENDA, SEM_VENDA e INATIVIDADE. Falta separar "cliente recusou" de "vendedora suspendeu"                                                              |
| MEL-16 | Abrir e consultar conversas    | **N**   | O cliente do WAHA só tem a prévia da última mensagem. Não há endpoint de mensagens                                                                                      |
| MEL-17 | Captura em celular corporativo | **P**   | A migração 39 criou as colunas e **diz na própria cabeça** que não liga ao WAHA, não roteia e não captura. Zero código usa                                              |
| MEL-18 | Papéis de Helena e Anastácia   | **P/C** | Os nomes no sistema são **Elena** e **Anastasia**, e os papéis estão invertidos no documento                                                                            |

## Consignações, Ocorrências, Clientes

|        | Item                          |       | O que existe hoje                                                                                                 |
| ------ | ----------------------------- | ----- | ----------------------------------------------------------------------------------------------------------------- |
| MEL-19 | Módulo de Consignações        | **T** | Módulo, tela, status, saldo, e separação estrutural das vendas: analytics nunca lê consignação                    |
| MEL-20 | Módulo de Ocorrências         | **P** | Existe e exige produto. **Não** exige cliente, **não** aceita foto, e a consulta histórica é só contagem por tipo |
| MEL-21 | Lista de clientes anonimizada | **P** | A tela é agregada e não mostra nome nenhum — mas também não é uma lista. E os nomes **não são cifrados**          |

Sobre o MEL-21, uma correção que vale registrar: a tarja diz *"dados pessoais
protegidos (AES-256)"*, e isso é verdade para **telefone, e-mail e observações**.
O campo `nome` é `varchar` puro, sem transformer. A frase promete mais do que
entrega.

## Interface, permissões e segurança

| | Item | | O que existe hoje |
|---|---|---|---|
| MEL-22 | Permissões granulares por perfil | **T** | Tela de papéis com uma caixa por permissão, 36 chaves, criar e apagar papel |
| MEL-23 | Proteção de dados de terceiros | **P** | Vendas e o canal da Elena têm recorte completo. **O `GET /clientes` não tem recorte nenhum** |
| MEL-24 | Mensagens com opções em lista | **C** | As personas **proíbem** lista e markdown, de propósito |

## Integrações

|        | Item                            |       | O que existe hoje                                                                                                                                                                 |
| ------ | ------------------------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MEL-25 | API de vendas para o Alessandro | **P** | `POST /vendas` por chave de API com escopo `vendas:write` existe, com tela de gestão de chaves. **Leitura não é exposta**: o escopo `vendas:read` existe e nenhuma rota o consome |
| MEL-26 | Limpeza da aba vendas           | **T** | Feito em 03/09: 1.269 vendas de seed apagadas (2.568 itens e 1.269 pagamentos por cascade) e a view `vendedoras_metricas` refeita. "Sala de Vendas" = a **tela de Vendas** — confirmado pelo Lucas em 04/09 |

O MEL-25 é P0 no documento, com prazo de D+1 — e a resposta depende do **sentido**
da integração, que a pendência nº 2 do próprio documento admite não saber. Se o
Alessandro **manda** venda para cá, está pronto. Se ele **lê** daqui, não existe
rota nenhuma para chave de API.

---

## Os cinco achados que mudam o planejamento

### 1. O MEL-22 e o MEL-23 se anulam

O `GET /clientes` tem só `@Permissions('clientes:read')`. Sem recorte por vendedora,
e sem um `EscopoClientesService` como o que vendas tem. O que protege hoje é apenas
o papel VENDEDORA não ter aquela permissão.

**Marcar `clientes:read` para VENDEDORA na tela de papéis — exatamente o que o
MEL-22 celebra como funcionalidade — entrega a base inteira de clientes, com
telefone, e-mail e limite de crédito, para qualquer vendedora.**

A tela granular é a arma que dispara o buraco. Fazer o MEL-22 antes do MEL-23
inverte a ordem segura.

### 2. O MEL-13 parte de um engano sobre o que é Demandas

Demandas é a **fila de chamado interno para a equipe técnica** — relatório, ajuste,
dúvida — aberta pela própria usuária ou registrada pela Anastasia quando não resolve
na conversa. Não é atendimento a cliente. Fundir com o WhatsApp destrói uma função
que não tem substituto.

E a tela que o documento descreve como "Acompanhamento" **já existe com outro
nome**: `/admin/auditoria`, com a timeline do MEL-14 dentro. O pedido, no fundo,
pode ser só renomear e mover na navegação.

### 3. O MEL-16 depende do MEL-17, e o MEL-17 tem um custo que o documento não menciona

A conversa cliente–vendedora acontece entre **dois telefones pessoais**, fora de
qualquer sistema nosso. Está escrito no código, no cabeçalho da auditoria.

Conectar o corporativo ao WAHA resolve — mas a sessão do WAHA lê a **conta inteira**.
Toda conversa daquele número passa a ser legível, não só a de trabalho. É decisão de
política, não de implementação, e a migração 39 já registrou esse mesmo alerta ao
criar as colunas sem ligar nada.

### 4. O MEL-09 tem o teto na geração, não na exportação

A exportação já entrega o arquivo byte a byte, sem recomprimir. O que limita é a
imagem **nascer 1024×1024 PNG** — resolução de tela, não de gráfica. Mexer na
exportação não resolve; o que resolve é gerar maior, e isso muda custo por imagem.

### 5. O MEL-24 pede o contrário de uma decisão tomada

As três personas proíbem lista, marcador e markdown, por escrito, porque o WhatsApp
renderiza mal e o tom fica de robô. A única exceção já existente é a escolha de peça
do catálogo, que **já é** numerada.

Se a decisão vai mudar, que mude por escrito — não entre como "ajuste de
usabilidade" P3.

---

## Antes de planejar sprint

Três perguntas que o batimento levanta, e que não estão na lista de pendências do
documento:

1. **Quem aprova a foto?** Hoje é quem fotografou, na conversa. O MEL-06 quer o
   marketing, na tela. São fluxos diferentes, e o segundo acrescenta uma espera
   entre fotografar e publicar.
2. **O que fazer com Demandas?** Ela não é o que o documento pensa. Some, fica, ou
   vira outra coisa?
3. **O que é "Sala de Vendas"?** O nome não existe no sistema.

E as seis pendências que o próprio documento lista continuam de pé — sobretudo a
nº 2, o sentido da integração com o Alessandro, que é o que decide se o item P0
está pronto ou nem começou.

---

## Documentos relacionados

- [[2026-08-20 — Montador de Catálogo — Levantamento]] — o levantamento que originou
  o módulo de catálogo, com a regra de preço e os quatro templates
- [[2026-08-28]] — a decisão de aprovar a foto **no WhatsApp**, que o MEL-06 revisita
- [[2026-09-01]] — a troca de "Gerar catálogo com IA" por "Montar catálogo em PDF",
  pelo mesmo motivo que o MEL-12 pede: rótulo não pode mentir sobre o mecanismo

---

# Verificação em produção — 03/09, à tarde

O batimento acima foi feito lendo código. Depois disso a base de produção foi
consultada, e ela contou uma história que o código não contava.

## Os números reais

| tabela | linhas | seed | observação |
|---|---|---|---|
| produtos | 6.939 | **0** | reais, entrando desde 05/08 |
| clientes | 1.090 | **0** | reais |
| vendedoras | 23 | 0 | inclui `VD02-TESTE`, criada à mão |
| **vendas** | 1.269 | **1.269** | **a tabela inteira era de mentira** |
| metas | 5 | 4 | a quinta era teste digitado à mão |

## O que isso revelou

**O painel de vendas nunca mostrou uma venda real.** Faturamento, ticket médio,
top vendedoras, gráfico de receita — tudo o que a Faby via desde sempre saiu do
`dev_seed.sql`. Produto entra desde 05/08, cliente e vendedora também. Só venda
que nunca chegou.

Isso muda o **MEL-25** de "entregar a API" para "cobrar quem manda": a API
existe (`POST /vendas`, chave + escopo `vendas:write`), e nunca foi usada.

**A `erp_eventos_processados` está vazia.** Nenhum evento, de tipo nenhum. Então
os 6.939 produtos **não entraram pelo webhook do Safira** — vieram por outro
caminho, provavelmente o `POST /produtos` com chave de API. O webhook do ERP
nunca foi exercitado.

## A limpeza

```sql
DELETE FROM vendas WHERE codigo_erp LIKE 'SEED-V%';   -- 1.269
-- itens_venda (2.568) e pagamentos_venda (1.269) caíram por cascata
DELETE FROM metas;                                     -- 5
REFRESH MATERIALIZED VIEW CONCURRENTLY vendedoras_metricas;
```

O `REFRESH` não é detalhe: `vendedoras_metricas` é view materializada sobre
`vendas`. Sem atualizar, os números do seed continuariam vivos na tela depois de
apagados do banco — e alguém concluiria que o delete não funcionou.

**Consequência a comunicar:** Vendas e Analytics ficam vazias. É o estado
honesto, mas quem abrir sem saber vai achar que quebrou.

Isso fecha o **MEL-26**. Ele era literalmente "a tabela de vendas é seed".

## Achados que mudam a aba Produtos

**As imagens existem, mas só em HTTP.**
`http://www.conexatecnologia.com/clientes/ATJewel/{codigo_erp}.png` responde 200
com PNG de 11 KB a 143 KB. **HTTPS não responde nada.** Como o painel é HTTPS, o
navegador tenta promover, falha, e bloqueia a imagem.

> **Miniatura de produto exige proxy no back.** Não existe caminho de "só
> exibir": em produção dariam 6.939 quadrados quebrados. O servidor busca em
> HTTP (o que ele pode) e serve pelo HTTPS do painel.

**A URL é derivável do código.** Sempre `{codigo_erp}.png`. Ou seja, a coluna
`foto_url` guarda o que dá para calcular — e o defeito conhecido do upsert do
ERP (que zera `foto_url` quando o Safira não manda) vira irrelevante.

**A foto parou de vir em 28/08.** As cargas grandes (05/08, 07/08, 20/08) vieram
com ~95% de cobertura. Tudo que entrou de 28/08 para cá — 28 peças em quatro
dias — veio **sem foto nenhuma**, e a imagem não existe no servidor (404). Volume
pequeno, mas é justamente a peça nova, que é a que vai para catálogo novo.

**A coluna Fornecedor está vazia em 6.939 de 6.939.** `referencia_fornecedor` é
NULL em todas. A tabela de produtos tem uma coluna que só mostra "—".

## Um defeito em produção que ninguém tinha visto

O leitor de legenda do WhatsApp reconhece código **pela forma** — duas letras e
dígitos, o padrão `CO26185` tirado dos catálogos impressos. A base real tem
quatro famílias:

| formato | peças | o que acontece |
|---|---|---|
| `AA#####` (`AL24001`) | 5.919 | entende |
| outro | 611 | não entende |
| `A###AA` (`C072RCH`) | 354 | não entende |
| com hífen (`1-25-3A-2`) | 55 | **entende errado** |

**1.020 de 6.939 — uma peça em sete.** E a família com hífen é a pior: como
começa com dígito, o `1` ou o `25` de `1-25-3A-2` pode ser lido como **número de
catálogo**. A foto vai para a coleção errada, sem erro e sem pergunta.

**A correção não é ajustar o regex** — no dia em que a Conexa mudar o padrão de
novo, como já mudou com as fotos em 28/08, quebra outra vez. O banco sabe quais
códigos existem: extrair os pedaços da legenda e **consultar a tabela**. Resolve
as quatro famílias de uma vez, resolve a colisão do hífen (o token inteiro é
consumido antes de alguém ver o `1` dentro dele) e resolve as 611 "outro" sem
precisar saber como elas são.

**Custo aceito:** código ainda não sincronizado deixa de ser reconhecido — hoje
é aceito às cegas. A foto continua entrando: o fluxo já grava sem descritivo e
já avisa quem mandou.

---

# A fila

> Atualizada no fim de **04/09**. **Dezenove dos vinte e seis itens estão
> fechados.** O que sobra são cinco travados numa pergunta só e uma validação.

1. **Conferir Vendas e Analytics vazias** — estado vazio é legítimo; quebrar não
   é. Divisão por zero e gráfico sem série são o risco. Já dá para conferir na
   produção, que está com zero vendas desde a limpeza de 03/09.
   *Nota de 04/09:* a barra do ranking de vendedoras tem guarda (`|| 1` no
   denominador). Não achei divisão desprotegida — pode ser só validação de olho.

**É a única coisa livre que sobrou.**

## Os cinco travados são um só assunto

**MEL-13, MEL-14, MEL-15, MEL-16 e MEL-17** são a mesma frente: ligar o WhatsApp
corporativo das vendedoras, ler as conversas, alimentar a timeline com elas e
tirar o desfecho do que a conversa diz. Demandas sai do menu no mesmo movimento.

**Todos destravam com uma resposta:** qual dos dois números recebe o QR Code. O
desenho está em [[2026-09-04 — Atendimento no WhatsApp — Desenho]], e a pergunta
está lá em destaque porque errá-la significa ler o celular pessoal da vendedora.

## O que fica fora do documento

- **CPF/CNPJ de cliente** — não existe coluna em lugar nenhum; a única do banco
  é `fornecedores.cpf_cnpj`. É migração nova mais a decisão de cifrar.
- **`produtos` não tem `empresa_id`** — se o ERP serve mais de uma empresa, as
  peças de todas caem na mesma tabela.
- **As duas personas invisíveis** na tela de Prompts. O Lucas optou por deixar.
- **A conta da Anthropic sem saldo** — derruba a Elena e a Anastasia.

### Feitos

- ~~**A aba Produtos**~~ — 04/09. MEL-01, MEL-02, MEL-03, MEL-04 e a coluna
  Fornecedor fora.
- ~~**Catálogo**~~ — 04/09. MEL-05 a MEL-12.
- ~~**Clientes + Papéis**~~ — 04/09. O buraco do `GET /clientes` fechado com
  `clientes:read_all` e um `EscopoClientesService`, mais a tabela e o cadastro
  de cliente.
- ~~**O leitor de legenda**~~ — 04/09. **E o desenho mudou depois de olhar os
  dados da produção**: a proposta que eu defendia havia dois dias estava errada
  em três pontos. Ver [[2026-09-04]], seção 7.

## Decisões que travam trabalho

- **`VD02-TESTE`** — vendedora de teste criada pelo Lucas. Fica ou sai? Aparece
  em todo seletor de vendedora do painel. Se sair, inativar é melhor que apagar:
  cliente, consignação e atendimento a referenciam.
- **Qual número recebe o QR Code** — a pergunta que trava MEL-13 a MEL-17 de uma
  vez. Os nomes das colunas estão **invertidos** entre o que o Lucas chama de
  interno/externo e o que a migração 39 chama; construir sem resolver isso pede o
  QR do celular PESSOAL da vendedora. Ver
  [[2026-09-04 — Atendimento no WhatsApp — Desenho]].
- **CPF/CNPJ de cliente** — não existe coluna. O Safira manda? Alguém precisa
  dele no painel, ou só na nota fiscal?

### Resolvidas em 04/09

- ~~Quem aprova a foto (MEL-06)~~ — o fluxo atual já é o que se queria.
- ~~Segregação Estoquista/Marketing (MEL-03)~~ — não existe essa segregação.
- ~~Alta qualidade na exportação (MEL-09)~~ — já é o melhor que temos; o teto é
  do modelo.
- ~~O que é "Sala de Vendas" (MEL-26)~~ — é a **tela de Vendas**, e o pedido era
  limpar os dados de seed. Feito em 03/09. Eu reabri esta dúvida em 04/09 por
  descuido: a resposta já estava escrita nesta mesma nota, na seção da limpeza.
- ~~O sentido da integração de vendas (MEL-25)~~ — **feito e entregue**,
  informado pelo Lucas em 04/09. Era o único P0 do documento.
- ~~O que fazer com Demandas (MEL-13)~~ — **desativar sem excluir**, e a tela
  vira o MEL-14. Decidido em 04/09; a execução depende da pergunta do QR Code.
- ~~Lista nas mensagens (MEL-24)~~ — vale no canal interno. Feito.
- ~~Mascarar o nome do cliente (MEL-21)~~ — não precisa: o recorte por carteira
  já garante que a vendedora só vê os dela.
