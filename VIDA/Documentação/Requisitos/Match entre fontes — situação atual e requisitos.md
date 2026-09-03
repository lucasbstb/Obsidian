# Match entre fontes — situação atual e requisitos

> Documento de requisitos — levantamento, não implementação.
> Fonte: slides "FLUXO DE ALIMENTAÇÃO VIDA Rio" (Caio Torres) + ata da reunião
> com Lucas Barbosa, Yerlon Magalhães e Caio Torres.
> Relacionados: [[Arquitetura e mapa do sistema]],
> [[Importar a planilha do ISP]], [[2026-09-02]].

> ⚠️ **Baseline: `release/1.0.0`**, não `main`.
> A `main` está **76 commits atrás** no back e **89** no front — e é a versão
> **de Fortaleza**, anterior à adaptação para o Rio. A primeira versão deste
> documento comparou contra ela e estava errada. Reescrito em 02/09/2026.

---

# 1. Onde o código vive

| Repositório | Branch de trabalho | Além da `main` | Último commit |
|---|---|---|---|
| `vida-rio-api` | **`release/1.0.0`** | +76 | 2026-06-17 |
| `VIDA-RIO` | **`release/1.0.0`** | +89 | 2026-06-17 |

Outras branches do back, todas com trabalho relevante:

```
feature/import-cbrj                              +66  · 2026-06-17
develop                                          +61  · 2026-04-22
bugfix/import-cor-data                           +60  · 2026-04-22
feature/implementation-cet-data-import-VDS-879   +35  · 2026-01-28
bugfix/resolve-issue-importing-isp-data          +16  · 2025-11-11
```

**Como identificar visualmente:** a `main` mostra "Fortaleza Prefeitura" em verde
na tela de login; a `release/1.0.0` mostra "Prefeitura Rio" em azul marinho, com
Transitar e STB no rodapé. O ambiente publicado em `10.29.0.167:3000` roda a
release.

---

# 2. O que JÁ está implementado

## 2.1 Importadores na API — com fila

```
POST /api/incident/importer/isp
POST /api/incident/importer/cor
POST /api/incident/importer/cbrj
GET  /api/incident/importer/template/isp
GET  /api/incident/importer/template/cor
```

```
src/incident/services/incident-importer-services/
├── importer-isp.service.ts
├── importer-cbmrj.service.ts
├── importer-cet.service.ts
├── isp-cor-data.service.ts          ← registra o par ISP × fonte
├── normalizaion-data-isp.ts
├── headers-isp-to-file-import.ts
└── export-templates-for-imports.service.ts

src/queues/processors/
├── incident-cor-import.processor.ts
└── incident-cbmrj-import.processor.ts
```

E o front tem os botões correspondentes na tela de Importadores.

> Isso responde duas coisas que ficaram em aberto: a ata estava certa ("a COR é
> carregada pela plataforma"), e o **Redis não é dependência morta** — alimenta
> as filas de importação assíncrona.

> E levanta outra: **o script Python `migracao.py` ainda é necessário?** Existe
> `importer-isp.service.ts` na API. Confirmar com o Yerlon antes de continuar
> rodando o script à mão.

## 2.2 Colunas por fonte

```typescript
@Column({ name: 'accident_code' }) accidentCode: string;
@Column({ name: 'isp_code' })      ispCode: string;
@Column({ name: 'cor_code' })      corCode: string;
@Column({ name: 'cbrj_code', nullable: true }) cbrjCode: string | null;
```

Falta apenas **`cet_code`** — está na branch
`feature/implementation-cet-data-import-VDS-879`.

## 2.3 Ação no MATCH — o enriquecimento existe

`saveTrafficIncidentWhenSimilar()` no importador CBMRJ:

```typescript
await queryRunner.manager.update(TrafficIncident, incident.id, {
  occurredAt:        trafficIncidentEntity.occurredAt,          // ajustar data/horário
  natureId:          trafficIncidentEntity.natureId,            // ajustar natureza
  affectedElementId: trafficIncidentEntity.affectedElementId,   // elemento atingido
  cbrjCode:          trafficIncidentEntity.cbrjCode,            // ID da fonte
  isDuplicated:      true,
});
```

Mais remoção dos veículos antigos e inserção dos novos, e o registro do par via
`ispCorDataService.createDuplicityIspAndCorData()`.

> **É exatamente a lista de "AÇÕES QUANDO MATCH" dos slides.** O comportamento de
> arquivar-e-perguntar da `main` foi substituído por enriquecimento automático.

## 2.4 Ação no NÃO MATCH — verificável nos dados

Os 11.180 registros da COR no banco de dev provam a regra rodando:

| Regra do slide | Evidência |
|---|---|
| COR como fonte primária | `first_source_information_id = 9` |
| Vítimas = **mesma quantidade de veículos** | **17.871 = 17.871**, exato |
| Se "com vítima" → 1ª lesionada | 5.099 vítimas com severidade **Leve** |
| Se "atropelamento" → 1ª pedestre | 358 vítimas do tipo **Pedestre** |
| Demais NI | 17.513 e 12.772 "Não informado" |
| **Não associar a veículos** | **17.871 de 17.871 sem vínculo** — 100% |

Os números fecham: 5.099 + 12.772 = 17.871.

## 2.5 Filtro por fonte — a "demanda extra"

Implementado com semântica de **conjunto exato**:

```typescript
if (count === 1 && hasISP) {
  queryBuilder.andWhere(
    `firstSourceInformationId = :ispId AND ${NO_COR} AND ${NO_CBRJ}`, { ispId: 2 });
}
```

Ids de fonte no domínio: **ISP = 2 · COR = 9 · CBMRJ = 12**.

> ⚠️ O banco de dev (`10.29.0.167`) tem só `2 ISP`, `9 COR`, `11 Não informado` —
> **falta o 12 (CBMRJ)**. O domínio está desatualizado em relação ao código.

Falta o **CET** na lista.

## 2.6 A regra de match — existem DUAS, uma por fonte

Levantado com precisão em 02/09/2026, lendo `find-similar-record.ts` e
`incident.service.ts` na `release/1.0.0`.

| Caminho | Checa duplicidade? | Regra | Ação |
|---|---|---|---|
| **ISP** (importador) | ❌ não checa | — | é a **fonte primária**, sempre insere |
| **COR** (importador) | ✅ `duplicityCheck` | ±3h + **geografia ≤ 500 m** | enriquece |
| **CBRJ** (importador) | ✅ `duplicityCheckCbmrj` | ±3h + **endereço normalizado, Dice ≥ 0,8** | enriquece |
| **Cadastro manual** | ❌ **não checa mais** | — | insere direto |
| **Tela de Duplicidades** | ✅ `duplicityCheck` | ±3h + geografia ≤ 500 m | resolução humana |

> **A COR casa por distância; o CBRJ casa por nome de via.** São critérios
> diferentes para o mesmo problema, e nenhum dos dois usa bairro.

### `duplicityCheck` — usado pela COR

```typescript
occurredAt BETWEEN (ocorrência - 3h) AND (ocorrência + 3h)
  AND wasRemoved IN (false, NULL)
  AND wasDuplicityResolved = false
→ compareIncidentsByLocation()   // Haversine, mantém os que estão a < 500 m
```

O raio foi ajustado três vezes ao longo do projeto:

```
92fc874  inicial ........  100 m
2547909  integração Mongo  1000 m
62ef0eb  ajuste corCode ..  500 m   ← atual
```

Não é código abandonado: é parâmetro que alguém vem calibrando.

### `duplicityCheckCbmrj` — usado pelo CBRJ

Mesma janela de ±3h, mas compara **texto**:

```typescript
normalizeAddress(input.firstAddress)
→ para cada candidato: compareTwoStrings(normA, normB) >= 0.8
```

Três limitações importantes:

1. **Só usa `first_address`.** O `second_address` saiu da comparação — na `main`
   ele era considerado. Cruzamentos perdem metade da chave
2. **Não usa bairro**
3. **Não usa geografia** — mesmo com `geom` preenchido em 100% dos registros

### O cadastro manual deixou de checar

Na `main`, `createIncident` e `updateTrafficIncident` rodavam `duplicityCheck` e,
ao encontrar, marcavam o novo registro com `hasArchived: true, isDuplicated:
true` — arquivando-o até um humano resolver.

**Na release essas chamadas não existem mais.** A verificação migrou inteira para
os importadores.

> Efeito colateral: um sinistro cadastrado pela tela **não é mais confrontado**
> com o que já existe. Verificar se foi decisão ou perda acidental na reescrita.

## 2.7 Normalização de endereço — parcial

`normalizeAddress()` em `find-similar-record.ts`, que foi **reescrito** na
release (43 inserções, 56 remoções):

| Passo do slide | Implementado |
|---|---|
| 1. Converter para maiúsculas | ✅ `.toUpperCase()` |
| 2. Remover acentos | ✅ `.normalize('NFD')` |
| 3. Ajustar após vírgula ou 1º traço | ❌ |
| 4. Expandir abreviações | ⚠️ **8 entradas** — AV., AV, R., ESTR., PRES., MIN., EMB., ENG. |
| 5. Remover espaços duplicados | ✅ `.replace(/\s+/g, ' ')` |
| 6. Tratar cruzamentos (Log 2) | ❌ |
| 7. Dicionário de equivalências | ❌ |

E a busca por geografia (<100 m) saiu — hoje o `findSimilarRecord` casa por
endereço normalizado.

---

# 2b. Conformidade ISP × COR

Análise linha a linha em 02/09/2026, contra a `release/1.0.0`.

## O ISP não precisa de tratamento — ele **é** o formato de referência

```typescript
firstAddress  = details['logradouro'];
secondAddress = details['intersecao'];
reference     = details['ref_numerica'];
neighborhood  = details['bairro'];
```

Direto, sem parsing. É a fonte primária: **não faz checagem de duplicidade**,
sempre insere.

### O importador do ISP da API resolve o bug do script Python

```typescript
// normalizaion-data-isp.ts
export const skinColorMapping = {
  'BRANCA': 1, 'ALBINA': 2, 'PRETA': 3, 'PARDA': 4,
  'AMARELA': 5, 'INDÍGENA': 6, 'SEM INFORMAÇÃO': 7,
};
```

Mapa explícito do vocabulário da planilha para os ids do domínio — e o mesmo para
`maritalStatusMapping` e `scholarityMapping`.

É exatamente o que falta no `migracao.py`, que usa
`LOWER(name) LIKE '%{cor}%'` e por isso jogou **todas as 47.760 vítimas** em
"Não informado". Ver [[Importar a planilha do ISP]].

**Outras duas vantagens do importador da API:**

| | Script Python | Importador da API |
|---|---|---|
| `accident_code` | `controle.split("-")[0]` → **91 códigos colididos, 1.716 sinistros (4,4%)** | `generateIdentificationCode()` → sem colisão |
| Cor / escolaridade / estado civil | `LIKE`, falha em 100% | mapa explícito |
| Agrupamento | `groupby("controle")` | `mountGroupIncidents` por `controle` |

> **Conclusão: o `migracao.py` está obsoleto.**

### Importar o ISP pela API — o que saber

| | |
|---|---|
| Tela | Importadores → botão "XLSX ISP" |
| Rota | `POST /api/incident/importer/isp` |
| Template | `GET /api/incident/importer/template/isp` |

⚠️ **Não há validação de cabeçalho.** O `headersIspFiles` só gera o template; o
importador lê por nome de coluna, e coluna ausente vira campo vazio.

A `Sinistros_ISP_2022-2025.xlsx` tem 26 das 28 colunas — faltam `evento` e
`dsc_local_fato`, que entrariam vazios. E o importador espera
`dsc_titulo_criminal`, que é o nome **novo**: foi feito para esse formato.

⚠️ **É síncrono, sem fila** (COR e CBRJ têm). 14 MB gerando ~38 mil sinistros
dentro de uma requisição HTTP provavelmente estoura o timeout do nginx (60s
padrão). Testar com um recorte pequeno antes.

⚠️ **O `controle` não é guardado.** O `accidentCode` é gerado e não vi o
`controle` ir para o `ispCode` — perde-se a chave que permitiria reconciliar com
a planilha depois. Confirmar com o Yerlon se é intencional.

---

## COR — as ações estão certas, o critério não

### ✅ Não match: implementado literalmente

```typescript
for(let i = 0; i < qtdVehicles; i++) {           // vítimas = qtd de veículos
  victimDTO.name = `Vítima ${i + 1}`;

  if((i === 0) && (severityId === INJURED)) {
    victimDTO.severityInjuryId = INJURED;         // 1ª lesionada se "com vítima"
  } else {
    victimDTO.severityInjuryId = NOT_REPORTED;    // demais NI
  }

  if((i === 0) && (natureId === HIT_BY_A_CAR)) {
    victimDTO.kindPersonId = 3;                   // 1ª pedestre se atropelamento
  }
}
```

O `vehicle_code` nunca é preenchido → "não associar vítimas a veículos".
Confirmado nos dados: **17.871 de 17.871 sem vínculo**.

### ✅ Match: implementado

```typescript
occurredAt         → ajustar data/horário
natureId           → ajustar natureza
affectedElementId  → elemento atingido
corCode            → adicionar ID da fonte COR
```

> O próprio slide diz: *"A lógica dos ajustes dos registros em casos de match não
> muda nada em relação ao que já está implementado"*. Essa parte é **documentação
> do que existe**, não pedido de mudança.

### 🔴 O critério diverge

| | Slide pede | Código faz |
|---|---|---|
| Critério 1 | Time ±3h | ✅ ±3h |
| Critério 2 | **Nome de via — ISP × COR tratado** | ❌ **geografia ≤ 500 m** |
| Critério 3* | Nome do bairro | ❌ não usa |

**A COR casa por distância, não por nome de rua.** É a divergência central.

### ⚠️ Uma divergência menor

Os veículos são **substituídos**, não adicionados:

```typescript
if (vehicles.length > 0) await queryRunner.manager.remove(Vehicle, vehicles);
...
await queryRunner.manager.save(Vehicle, newVehicles);
```

O slide diz "Adicionar veículos". Como ele também diz que o match "não muda
nada", pode ser comportamento aceito — confirmar com o Caio.

---

## O tratamento de endereço está mais completo do que parecia

O importador da COR já faz parte do trabalho, no momento da importação:

```typescript
const [streets, others] = row['localizacao'].split(',');   // passo 3: após a vírgula
if(streets.includes('-')) {
  firstAddress = streets.slice(0, dashIndex);              // passo 3: após o 1º traço
  reference    = streets.slice(dashIndex + 1);
} else {
  const [firstStreet, secondStreet] = streets.split(' x '); // passo 6: cruzamento
  firstAddress  = firstStreet;
  secondAddress = secondStreet;
}
```

| Passo do slide | Onde está | Situação |
|---|---|---|
| 1. Maiúsculas | `normalizeAddress` | ✅ |
| 2. Remover acentos | `normalizeAddress` | ✅ |
| 3. Após vírgula ou 1º traço | **importador COR** | ✅ |
| 4. Expandir abreviações | `normalizeAddress` | ⚠️ **8 de ~40** |
| 5. Espaços duplicados | `normalizeAddress` | ✅ |
| 6. Tratar cruzamentos | **importador COR** | ⚠️ **só ` x `**, de 14 conectores |
| 7. Dicionário de equivalências | — | ❌ |

> **Mas há uma separação que os slides não preveem.** Os passos 3 e 6 rodam no
> **importador**, produzindo `first_address`/`second_address`. Os passos 1, 2, 4
> e 5 rodam no **match**, dentro do `normalizeAddress`. São dois tratamentos
> distintos; o slide descreve um pipeline só, gerando `LOC_TRATADO`.

---

## FECHAMENTO — o que falta em ISP × COR

### ISP: nada

É a fonte primária. Não faz match, sempre insere. Os slides não pedem
pré-tratamento nem critério para ela.

Único ponto a confirmar: o `controle` da planilha **não é guardado** — o
`accidentCode` é gerado e o `ispCode` não recebe o `controle`. Perde-se a chave
de reconciliação com a origem. Intencional?

### COR: quatro lacunas

| # | Slides pedem | Situação | Tamanho |
|---|---|---|---|
| **1** | **Dedup interna** — "deixar apenas o último registro" | ✅ **feito** (`de73700`) | pequeno |
| **2** | Critério 2: **nome de via tratado** | ✅ **feito** (`de73700`) | pequeno |
| **3** | Critério 3: **bairro** | ✅ **feito** (`de73700`) | pequeno |
| **4** | **`LOC_TRATADO`** — normalização completa | ⚠️ 5 de 7 passos, 2 parciais | médio |

> **Itens 1, 2 e 3 implementados em 02/09/2026**, na branch `lucasdev`. Detalhe
> e medição de impacto em [[2026-09-02]], seção 8.
>
> A dedup ficou por **`Id_ocorrencia`**, não por hash de conteúdo como no CBMRJ:
> a planilha da COR traz a mesma ocorrência em estágios diferentes, então o
> conteúdo muda e o hash não pegaria. O bairro passou a usar a **mesma
> normalização do endereço** — sem isso, 29% dos registros da COR ficariam sem
> par só por acento e caixa (`MARE` no ISP × `Maré` na COR).
>
> ⚠️ **Abriu uma decisão nova.** Em via arterial longa, nome de via + bairro
> produzem match a mais de 10 km (Av. Brasil tem 58 km). Ver **RF-07** abaixo.

> **A dedup interna é a mais irônica:** os slides citam a COR como origem da
> regra ("Pré-Etapa 2: Tratar duplicidades internas da FONTE COR"), e é
> justamente ela que não tem. O CBMRJ implementou
> (`buildDeduplicationMap` + `hashRowContent`).

### O detalhe do item 4

| Passo | Onde | Situação |
|---|---|---|
| 1. Maiúsculas | `normalizeAddress` | ✅ |
| 2. Remover acentos | `normalizeAddress` | ✅ |
| 3. Após vírgula ou 1º traço | importador COR | ✅ |
| 4. Expandir abreviações | `normalizeAddress` | ⚠️ **8 de ~40** |
| 5. Espaços duplicados | `normalizeAddress` | ✅ |
| 6. Tratar cruzamentos | importador COR | ⚠️ **1 conector de 14** |
| 7. Dicionário de equivalências | — | ❌ |

Os itens 4, 6 e 7 são **dado, não lógica** — cabem numa tabela de configuração.
É onde os slides gastam a maior parte das páginas, e o que exige **curadoria**:
o próprio slide avisa que alias só entra depois de confirmação em fonte oficial.

### Já pronto, sem nada a fazer

- **Todas as ações no match** — data/hora, natureza, elemento atingido, ID da COR
- **Todas as ações no não match** — 12 regras, implementadas literalmente e
  comprovadas nos dados (17.871 = 17.871, 5.099 Leve, 358 Pedestre, 100% sem
  vínculo a veículo)
- A janela de **±3h**

### Em uma frase

> Na COR, **as ações estão prontas**. Falta a **dedup interna** e trocar o
> **critério de detecção** de geografia para endereço tratado + bairro.

Os itens 1, 2 e 3 são de poucas linhas cada. O **4 é o trabalho de verdade** — e
é o mesmo que o CBRJ vai precisar. Vale fazer uma vez, compartilhado.

# 3. O que AINDA falta

Atualizado em **02/09/2026**, depois de receber os slides e implementar.

| # | Item | Situação |
|---|---|---|
| 1 | **Passo 3** — ajustar tudo após vírgula ou 1º traço | ✅ `3ed331f` |
| 2 | **Passo 6** — tratar cruzamentos | ✅ `3ed331f` |
| 3 | **Passo 7** — dicionário de equivalências | ✅ `3ed331f` |
| 4 | **Dicionários completos** — 4 dicionários, ~60 entradas | ✅ `3ed331f` |
| 5 | **Parser de cruzamento** — 14 conectores + 7 regras | ✅ `3ed331f` |
| 6 | **Chave de cruzamento ordenada** (`A x B` = `B x A`) | ✅ `3ed331f` |
| 7 | **`location_raw`** — preservar a frase original | ✅ `3ed331f` |
| 8 | **Critério de bairro** no match | ✅ `de73700` |
| 9 | **CET** — importador, `cet_code` e entrada no filtro | ⏳ em branch separada |
| 10 | **SMS (Saúde)** — "fonte complementar 4" | ❌ sem etapa detalhada nos slides |
| 11 | **Limpeza mecânica** — preservar grafia canônica do ISP para exibição | ✅ `3ed331f` |

Sobram o **9** e o **10**, que não são tratamento de endereço: um é integrar uma
branch parada, o outro não tem etapa detalhada nos slides.

## Onde o código ficou

```
src/utils/address/
├── dictionaries.ts        ← os 4 dicionários + os 14 conectores, como DADO
├── normalize-location.ts  ← passos 1-5 e 7, mais a limpeza mecânica
└── parse-intersection.ts  ← passo 6 e as 7 regras de interseção
```

Branch `lucasdev`, commits `3ed331f` e `4b2dc08`.

## Duas contradições dos slides, e como foram resolvidas

**Passo 3 × limpeza mecânica.** O passo 3 manda "ajustar tudo após a vírgula ou
1º traço". A tabela de limpeza mecânica diz que o hífen semântico de
`Rio–Niterói` e `Grajaú–Jacarepaguá` **não pode ser corte automático**. Lidos
literalmente, se contradizem.

> **Resolvido cortando só traço cercado de espaço.** É o que reconcilia os dois —
> e é exatamente o bug que produziu os 15 registros `PONTE RIO` na base, do
> importador antigo cortando "Ponte Rio-Niterói" no hífen.

**Chave da dedup interna.** O slide diz "deixar apenas o último registro" sem
dizer o último *de quê*. O exemplo mostra `COR2501810`, `COR2501811` e
`COR2501813` sendo deduplicados — **os `Id_ocorrencia` são diferentes**.

> **A chave é o conteúdo**, não o id: data + `pop` + `titulo` + `localizacao` +
> `bairro` + lat + long. Da `abertura` entra só a data, porque a hora varia entre
> as linhas da mesma ocorrência — incluí-la faria nada colapsar, e usar a
> `abertura` inteira faria ocorrências de dias diferentes se fundirem.
>
> Primeiro implementei pelo `Id_ocorrencia` e estava errado. Corrigido em
> `3ed331f`.

---

# 4. Requisitos derivados

## Normalização — todos implementados em 02/09/2026

- ~~**RF-01** — passos 3, 6 e 7~~ ✅ `3ed331f`
- ~~**RF-02** — dicionários como **dado**, para curadoria sem deploy~~ ✅ `3ed331f`
- ~~**RF-03** — parser de cruzamento: 14 conectores, 7 regras~~ ✅ `3ed331f`
- ~~**RF-04** — chave de cruzamento ordenada~~ ✅ `3ed331f`
- ~~**RF-05** — preservar a grafia original, separada da chave de busca~~ ✅ `3ed331f`

> O `original_address` estava **vazio nos 11.180 registros da COR** porque o
> `mapCreateTrafficIncidentDtoToEntity` fixava `originalAddress = null` e
> `hasIntersection = false`. Os dois passaram a vir do DTO.

## Match

- ~~**RF-06** — bairro como critério~~ ✅ `de73700`
- **RF-07** — proximidade geográfica: **feito no escopo que os slides pedem**
  (`4b2dc08`), que é menor do que a proposta anterior deste documento.

### O que os slides autorizam em geografia

| Onde | Texto |
|---|---|
| Transolímpica | "**Exigir coordenada ou segmento**; não aplicar como equivalência global" |
| Autoestrada Grajaú | "**Exigir validação espacial**" |
| Critérios de match (slides 11 e 12) | Time ±3h · Nome de via · Nome do bairro |

Geografia **não é critério de match** nos slides. Ela aparece só condicionando
duas entradas do dicionário de equivalências. Foi assim que ficou: quando o nome
direto não casa, tenta a identidade condicionada e **só aceita se as duas
coordenadas existirem e estiverem a menos de 500 m**. Sem coordenada, não aplica.

> O raio de **500 m não vem dos slides**. Adotado por ser o único parâmetro
> geográfico com histórico no projeto (já foi 100 m, 1000 m, hoje 500 m).
> **Confirmar com o Caio.**

### ⚠️ O que a medição mostra, e que os slides não previram

Numa amostra de 500 sinistros da COR, o critério dos slides produz 65 matches.
A distribuição das distâncias:

| <1km | 1-2km | 2-5km | 5-10km | **>10km** |
|---|---|---|---|---|
| 19 | 16 | 18 | 2 | **20** |

Os 20 acima de 10 km são casos assim:

```
Av Brasil | (bairro vazio)  ↔  AVENIDA BRASIL | VILA MILITAR      10.503 m
```

**A Avenida Brasil tem 58 km.** Dois sinistros nela, com 3h de janela e 10 km de
distância, quase certamente não são o mesmo evento.

> Isso **não é defeito da implementação** — é o resultado correto da regra como o
> documento a define. Em via arterial longa, nome de via + bairro não bastam, e
> quando o bairro vem vazio some a única coisa que segurava.
>
> **Decisão do Caio.** Uma trava geográfica geral resolveria, mas não está nos
> slides, e por isso não foi feita. Ver pergunta 9.

### O que ainda não casa, e por quê

Depois do pipeline completo, registros da COR sem nenhum par possível no ISP
caem de **1.880 (17,0%) para 327 (3,0%)**. Do que sobra:

| Via | Registros | Por quê |
|---|---|---|
| `AUTOESTRADA GRAJAU` | 45 | equivalência condicionada — aguarda validação espacial |
| `ELEVADO ENG. RUFINO DE A. PIZARRO` | 34 | `A.` → ALMEIDA não está em nenhum dicionário do slide |
| `ESTRADA GRAJAU/JPA` | 29 | equivalência condicionada |
| `PONTE RIO` | 15 | dano do importador antigo, que cortava "Ponte Rio-Niterói" no hífen |

## Fontes

- **RF-08** — `cet_code` e CET no filtro (integrar a branch)
- **RF-09** — Cadastrar CBMRJ (id 12) no domínio do banco de dev
- **RF-10** — Definir o escopo do SMS

---

# 5. Perguntas em aberto

Já respondidas pelo código, e por isso removidas desta lista: o critério de nome
de via (Dice ≥ 0,8 sobre texto normalizado), a proximidade geográfica (não foi
removida — virou 500 m e é o critério da COR) e o que roda em produção
(`release/1.0.0`, `da3e19e`/`2da46ef`).

O que sobra depende de decisão de negócio ou de conversa com o time.

## Para o Caio

**1. São dois ou três critérios?**
O slide diz "os DOIS critérios" mas lista três, com asterisco no bairro. E o
**bairro não é usado por nenhuma das duas regras atuais**, embora esteja
preenchido em 99,9% da base.

**2. COR e CBRJ devem usar o mesmo critério?**
Hoje a **COR casa por distância (500 m)** e o **CBRJ por nome de via
(Dice ≥ 0,8)**. Os slides descrevem a mesma regra para as duas. É convergência
desejada, ou a diferença é proposital?

**3. Qual o raio correto?**
O parâmetro já foi 100 m, depois 1000 m, hoje 500 m. Alguém está calibrando sem
critério documentado. E agora ele reaparece na validação espacial das
equivalências condicionadas, onde os slides pedem "exigir coordenada" mas não
dizem quanto.

**9. Via arterial longa — o critério dos slides basta?** *(nova, 02/09/2026)*
Implementado exatamente como o documento define, **20 de 65 matches numa amostra
ficam acima de 10 km** — `Av Brasil` casando com `Av Brasil` numa avenida de
58 km. Não é defeito da implementação; é o resultado da regra. Os slides não
tratam vias arteriais longas.

Três saídas possíveis, e a escolha é dele:

| | O que muda |
|---|---|
| Trava geográfica geral | reprova acima de um raio, quando há coordenada dos dois lados |
| Bairro obrigatório | resolve o caso de bairro vazio, mas perde match legítimo |
| Nada | aceita o falso positivo, com resolução humana na tela de Duplicidades |

> A terceira depende da tela de Duplicidades voltar a resolver — hoje as três
> rotas de resolução estão comentadas no backend.

**10. O `A.` de `ELEVADO ENG. RUFINO DE A. PIZARRO`.** *(nova)*
São 34 registros. O ISP escreve `ALMEIDA` por extenso. `A.` não está em nenhum
dos quatro dicionários, e expandir inicial solta é perigoso. Entra como
equivalência de via, ou fica assim?

**4. O `second_address` deveria voltar para a comparação?**
Saiu na reescrita. Em cruzamento, comparar só a primeira via perde metade da
identidade — e é justamente o que as regras de interseção dos slides pretendem
estruturar.

## Para o Yerlon

**5. O `migracao.py` ainda é o caminho para o ISP?**
Existe `importer-isp.service.ts` na API, com endpoint e template. Estamos
rodando script manual para algo que virou funcionalidade?

**6. O cadastro manual deixar de checar duplicidade foi decisão?**
Na `main` ele checava e arquivava; na release não checa mais. Intencional ou
perda na reescrita?

**7. Quando a branch do CET entra?**
`feature/implementation-cet-data-import-VDS-879` parada desde 28/01/2026,
enquanto a release avançou até junho. Quanto mais espera, mais caro integrar.

## Técnica, a resolver internamente

**8. O `accident_code` colidido afeta o match?**
Ver [[Importar a planilha do ISP]] — **91 códigos colididos**, atingindo **1.716
dos 38.592 sinistros (4,4%)** na carga feita pelo script Python. As regras de
match atuais **não** usam esse campo, mas os blocos de `UPDATE` do script usam.
E o horário depende deles: sem hora correta, o critério de ±3h compara
meia-noite com meia-noite.
