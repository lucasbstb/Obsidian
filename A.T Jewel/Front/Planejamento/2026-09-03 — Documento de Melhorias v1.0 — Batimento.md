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

---

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

| | Item | | O que existe hoje |
|---|---|---|---|
| MEL-05 | Textos em tooltip, linguagem formal | **N** | Não existe componente de tooltip; são oito parágrafos explicativos em tela |
| MEL-06 | Aprovação de fotos pelo marketing | **P/C** | O ciclo existe inteiro — seis status, só `APROVADA` entra no catálogo, o "X" tira e é reversível. Mas quem aprova é **quem fotografou, no WhatsApp** |
| MEL-07 | Várias imagens com observação individual | **P** | Várias imagens sim; observação por arquivo não — observações são linhas soltas, sem vínculo com o arquivo |
| MEL-08 | Cabeçalho padronizado | **P** | Nome, ID, contagem, criador e data existem. **Capa por catálogo não**: reaproveita a primeira referência, e no card da lista nunca mostra imagem |
| MEL-09 | Exportação em alta qualidade | **P** | ZIP com as fotos sem recompressão e o CSV do descritivo. Mas a imagem nasce **1024×1024** |
| MEL-10 | Upload de PDF como referência | **N** | Referência aceita só JPEG, PNG e WebP. PDF é aceito noutro lugar — no catálogo montado que volta do marketing |
| MEL-11 | Destaque do botão Anexar | **N** | Hoje é o elemento **menos** destacado do cartão: borda tracejada, cor apagada |
| MEL-12 | Trocar "ChatGpt" por "Agente" | **N** | Duas ocorrências visíveis, as duas no front. O back nunca nomeia o provedor |

## Atendimento e acompanhamento

| | Item | | O que existe hoje |
|---|---|---|---|
| MEL-13 | Aba única, eliminando Demandas | **P/C** | Parte de um engano — ver adiante |
| MEL-14 | Timeline visual por vendedora | **P** | Já existe em `/admin/auditoria`: cartão por vendedora e linha do tempo com hora, etapa, cliente, relato e próximo contato. Plota **episódios e relatos**, não conversas |
| MEL-15 | Regra de encerramento | **P** | `desfecho` aceita VENDA, SEM_VENDA e INATIVIDADE. Falta separar "cliente recusou" de "vendedora suspendeu" |
| MEL-16 | Abrir e consultar conversas | **N** | O cliente do WAHA só tem a prévia da última mensagem. Não há endpoint de mensagens |
| MEL-17 | Captura em celular corporativo | **P** | A migração 39 criou as colunas e **diz na própria cabeça** que não liga ao WAHA, não roteia e não captura. Zero código usa |
| MEL-18 | Papéis de Helena e Anastácia | **P/C** | Os nomes no sistema são **Elena** e **Anastasia**, e os papéis estão invertidos no documento |

## Consignações, Ocorrências, Clientes

| | Item | | O que existe hoje |
|---|---|---|---|
| MEL-19 | Módulo de Consignações | **T** | Módulo, tela, status, saldo, e separação estrutural das vendas: analytics nunca lê consignação |
| MEL-20 | Módulo de Ocorrências | **P** | Existe e exige produto. **Não** exige cliente, **não** aceita foto, e a consulta histórica é só contagem por tipo |
| MEL-21 | Lista de clientes anonimizada | **P** | A tela é agregada e não mostra nome nenhum — mas também não é uma lista. E os nomes **não são cifrados** |

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

| | Item | | O que existe hoje |
|---|---|---|---|
| MEL-25 | API de vendas para o Alessandro | **P** | `POST /vendas` por chave de API com escopo `vendas:write` existe, com tela de gestão de chaves. **Leitura não é exposta**: o escopo `vendas:read` existe e nenhuma rota o consome |
| MEL-26 | Limpeza da Sala de Vendas | **?** | "Sala de Vendas" não existe em lugar nenhum dos dois repositórios |

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

Em ordem, e o porquê da ordem.

1. **O leitor de legenda** — defeito em produção, uma peça em sete, e a variante
   silenciosa manda material para a coleção errada. Fica no back, é localizado, e
   o arquivo de spec da leitura de legenda já existe.
2. **A aba Produtos** — rota de imagem no back com cache (obrigatória, ver
   acima), coluna de miniatura, coluna de código com clique para copiar
   (**MEL-01 + MEL-02**, que colapsam na mesma coisa), e a coluna Fornecedor
   fora (**MEL-26**, no espírito).
3. **Conferir Vendas e Analytics vazias** — estado vazio é legítimo; quebrar não
   é. Divisão por zero e gráfico sem série são o risco.
4. **Clientes + Papéis** — o buraco do `GET /clientes`. Não é urgente por prazo:
   é urgente porque a tela de Papéis já está pronta e basta alguém marcar a
   caixa.
5. **Catálogo** — nove dos 26 itens, a aba mais pesada.

## Decisões que travam trabalho

- **`VD02-TESTE`** — vendedora de teste criada pelo Lucas. Fica ou sai? Aparece
  em todo seletor de vendedora do painel. Se sair, inativar é melhor que apagar:
  cliente, consignação e atendimento a referenciam.
- **Quem aprova a foto** (MEL-06) — hoje é quem fotografou, na conversa. O
  documento quer o marketing, na tela.
- **O que fazer com Demandas** (MEL-13) — ela não é o que o documento pensa.
- **O sentido da integração de vendas** (MEL-25) — decide se o P0 está pronto ou
  nem começou.
