# 2026-08-20 — Montador de Catálogo — Levantamento

Retrato do que se sabe hoje, antes de qualquer código. Nasceu da ideia do Lucas:
uma tela onde a vendedora anexa fotos a cada peça, e outra onde ela seleciona
peças e o sistema monta o catálogo.

Base do levantamento: os quatro catálogos reais em
`A.T Jewel/Documentação/Documentos/Modelo Coleção` — **ESMERALDA**,
**FATHER'S DAY**, **NEW IN** e **SUMMER II**.

---

## O que os catálogos são, de fato

| | |
|---|---|
| páginas | 10 a 15 por catálogo |
| imagens | 53 a 87 por catálogo (~6 por página) |
| peso | 17 a 143 MB — **exportados em resolução de gráfica** |
| formatos | **dois**: ESMERALDA em retrato 9:16 (story); os outros em paisagem 16:9 |

Não são listas de produto. São **peças de campanha sazonal**, com foto editorial
produzida (modelo, praia, ambiente) misturada a packshots.

> Avaliação corrigida no meio do levantamento: olhando só uma foto editorial
> solta, a conclusão foi "isso é trabalho de designer, não automatizável". Vendo
> as **páginas montadas**, a maioria é grade regular. É bem mais automatizável do
> que parecia. Vale como lição: imagem solta não mostra diagramação.

---

## Os quatro templates

| | template | descrição | automatizável? |
|---|---|---|---|
| **A** | Capa | editorial sangrando + título da coleção + logo | sim, simples |
| **B** | Grade de packshots | N×M peças, cada uma com bloco de texto abaixo. Observados 2×1, 3×2 e 4×2 — **é o mesmo template com N variável** | sim |
| **C** | Meia-a-meia | editorial ocupando metade + coluna com 2 ou 3 blocos de produto na outra | sim |
| **D** | Composição livre | peças dispostas em círculo, sobreposições | **não** — é arte |

**B e C cobrem quase todas as páginas de conteúdo.** As duas são flexbox simples.

---

## O bloco de produto é rigorosamente padronizado

```
CO26185 • COLAR ESMERALDA 2.10 CTS OB 18K 2.30G
R$43.920,00 à vista
10X de R$5.490,00
```

Sempre: **código • descrição com specs** / **preço à vista** / **parcelamento**.
Uma variação observada: `*preço sob consulta`.

### A regra de preço — verificada em 25 de 25

| parcelamento | preço à vista |
|---|---|
| 10X | total parcelado **× 0,80** |
| 6X | total parcelado **× 0,90** |

`10X de R$5.490` = R$54.900 → à vista R$43.920.
`6X de R$450` = R$2.700 → à vista R$2.430.

Bateu em **todos** os 25 produtos que aparecem nas páginas conferidas, sem uma
exceção.

**Consequência:** o sistema guarda **um** preço e calcula o outro. Não precisa de
dois campos.

**Pendente de confirmação com o Lucas:** o `valorVenda` que vem do ERP é o
parcelado (o "de") ou o à vista?

### Os códigos são o `codigoErp`

Prefixo de duas letras + número:

```
CO = colar     AN = anel      BR = brinco
PU = pulseira  PG = piercing  PI = pingente
```

É a mesma chave durável recomendada para amarrar as fotos — e já é o que aparece
impresso no catálogo. Ver a seção de armazenamento.

---

## O bloqueio: não existe foto de produto

O tipo `Produto` no front tem **11 campos e nenhum é imagem**:

```
id · codigoErp · categoria · familia · descricaoEtiqueta
referenciaFornecedor · tipoPedra · valorVenda · estoqueAtual
dataEntradaEstoque · ativo
```

`grep -rniE "imagem|foto|image|thumb"` em `lib/api/` inteiro: **zero**. Também
não existe upload em lugar nenhum do front, e o `apiFetch` só fala JSON.

**Isso é projeto de back antes de ser de front.**

### Amarrar a foto ao `codigoErp`, não ao `id`

Os produtos vêm **sincronizados do ERP Safira**. Se a foto for amarrada ao `id`
interno do CRM e uma resync recriar as linhas, **todas as fotos órfãm de uma
vez**. Amarrar ao `codigoErp` (o `CO26185`) é o que sobrevive à sincronização.

Barato de acertar agora, caríssimo depois.

---

## Armazenamento

### Espaço

Depende de uma decisão só: **redimensionar antes de subir**. Foto crua de celular
tem 3 a 6 MB; a 1600px no lado maior em JPEG 85 fica em **300 a 500 KB** — mais
que suficiente para os packshots dos catálogos.

| cenário | fotos | cru (5 MB) | redimensionado (400 KB) |
|---|---|---|---|
| base atual, 120 peças × 3 | 360 | 1,8 GB | **144 MB** |
| 500 peças × 3 | 1.500 | 7,5 GB | **600 MB** |
| 2.000 peças × 3 | 6.000 | 30 GB | **2,4 GB** |

Redimensionando, espaço **não é problema**. Sem redimensionar, vira — e é chato
de corrigir depois que o acervo existe.

### Onde

**Disco do VPS, em volume nomeado, servido pelo back.**

Descartados:
- **banco de dados** (`bytea`/base64) — incha o backup e deixa consulta lenta
- **S3/R2 agora** — são centenas de MB, não terabytes; traria credencial nova,
  CORS e mais uma coisa pra dar errado num deploy sem homolog
  (ver [[producao-roda-next-dev-bind-mount]])

Duas ressalvas que pesam mais que a escolha:

1. **Volume explícito.** Produção roda em container com bind mount. Recriar o
   container sem mapear o volume das fotos as apaga — e elas não estão no git
   nem no ERP.
2. **Backup é obrigatório.** O banco, em último caso, o ERP reconstrói. As fotos
   não. Perder o VPS significa refazer a sessão de fotos de centenas de peças.

Migrar para S3/R2 vira boa ideia quando o acervo passar de ~5 GB ou quando
quiser CDN. Aí é troca de uma camada, não de arquitetura.

### Acesso

**URL pública com nome não-adivinhável.** Packshot de joia não é segredo — vai
num catálogo que a vendedora manda pra cliente. Deixar atrás do JWT obrigaria o
gerador de PDF a baixar cada imagem autenticada e converter em base64,
complicando sem proteger nada de real.

---

## A foto — o ponto que muda o resultado

Os packshots dos catálogos são **fundo branco com sombra suave**. Uma foto de
celular tirada na bancada, com o fundo da loja, **não se parece com isso** — o
catálogo sairia com cara de mural.

Mas a correção é barata: **caixa de luz + fundo branco + celular no tripé** já
produz packshot utilizável. É captura padronizada, não tratamento. Treino de
vinte minutos com a vendedora, não contratação de estúdio.

As fotos **editoriais** (modelo, ambiente) continuam sendo produção — mas são
**por campanha**, não por peça. Uma sessão rende o ano.

---

## Divisão do trabalho

| o sistema faz | continua manual |
|---|---|
| acervo de packshot por peça | foto editorial da campanha |
| páginas **B** (grade) e **C** (meia-a-meia) a partir da seleção | copy de campanha ("our favorites, EMERALDS to be yours!") |
| bloco de produto com código, descrição e os dois preços | páginas tipo **D** — entram como imagem pronta |
| capa **A**: sobe a editorial, digita o título | |
| exporta nos **dois** formatos (9:16 e 16:9) | |

---

## Geração do PDF

**Recomendação: `@react-pdf/renderer`, gerando no navegador.**

Sai PDF **vetorial** — texto de verdade, fontes Marcellus e Manrope embutidas.
O template é componente React, e o preview na tela é o próprio componente: o que
se vê é o arquivo.

**Por que não o que já existe:** o `exportarElementoPdf` usa `html2canvas` →
PNG → `jsPDF`. Ele **rasteriza** — o PDF vira foto da tela. Serve pra fotografar
um painel de KPIs, que é pra onde foi feito. Pra catálogo dá texto borrado, não
selecionável, arquivo pesado e ~96 dpi, que gráfica recusa.

**Por que não Chromium headless** (`page.pdf()`), que daria fidelidade total de
CSS: produção roda `next dev` num bind mount, sem homolog. Botar Chromium
naquele container é mudança de infra testada direto no ar.

**O preço do react-pdf:** subconjunto de CSS. Flexbox sim, **grid não**. Para
capa, cabeçalho, grade de cards e rodapé, flexbox resolve.

---

## Ordem sugerida

1. **Back**: campo de foto amarrado ao `codigoErp`, endpoint de upload, volume
2. **Front**: tela de anexar fotos (reaproveita filtros e tabela de Produtos)
3. **Front**: templates B e C em `@react-pdf/renderer`, com preview
4. **Front**: capa (template A) e exportação nos dois formatos

O passo 1 é pré-requisito de tudo. Os passos 3 e 4 dependem de decidir os dois
formatos desde o começo — deixar para depois vira retrabalho de diagramação.

---

## Em aberto

- `valorVenda` é o parcelado ou o à vista?
- Quantas parcelas por peça: sempre 10X, ou varia? (vi 6X em dois colares)
- O "preço sob consulta" é um estado da peça ou decisão de quem monta?
- Quem monta o catálogo: a gestão ou a vendedora?
- O catálogo é para **WhatsApp** ou também para **gráfica**? (muda resolução,
  sangria e a conversa sobre CMYK)

---

## Documentos relacionados

- [[2026-08-11 — Integração ERP Safira — Levantamento de Requisitos]] — de onde
  vem o `codigoErp`
- [[Deploy do front em producao]] — por que o volume precisa ser explícito
