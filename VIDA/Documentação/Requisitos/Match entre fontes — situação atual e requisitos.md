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

## 2.6 Normalização de endereço — parcial

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

# 3. O que AINDA falta

Lista real, contra a `release/1.0.0`.

| # | Item | Situação |
|---|---|---|
| 1 | **Passo 3** — ajustar tudo após vírgula ou 1º traço | ❌ |
| 2 | **Passo 6** — tratar cruzamentos | ❌ |
| 3 | **Passo 7** — dicionário de equivalências (Linha Amarela → Av. Gov. Carlos Lacerda etc.) | ❌ |
| 4 | **Dicionários completos** — hoje 8 abreviações; os slides trazem ~40 entre vias, títulos e componentes | ⚠️ parcial |
| 5 | **Parser de cruzamento** — 14 conectores + 7 regras, alimentando `first_address`/`second_address` | ❌ |
| 6 | **Chave de cruzamento ordenada** (`A x B` = `B x A`) | ❌ |
| 7 | **`location_raw`** — preservar a frase original | ⚠️ existe `original_address`, verificar se cumpre |
| 8 | **Critério de bairro** no match | ❌ não usado no `findSimilarRecord` |
| 9 | **CET** — importador, `cet_code` e entrada no filtro | ⏳ em branch separada |
| 10 | **SMS (Saúde)** — "fonte complementar 4" | ❌ sem etapa detalhada nos slides |
| 11 | **Limpeza mecânica** — preservar grafia canônica do ISP para exibição | ❌ |

## O peso disso

Quase tudo que falta é **tratamento de endereço** — os itens 1 a 7. Não é
coincidência: é a parte dos slides com mais páginas (quatro dicionários, sete
regras de cruzamento, uma tabela de limpeza mecânica).

O motor de match, as colunas de fonte, o enriquecimento, os importadores e o
filtro **já existem**. O que falta é a qualidade da chave que alimenta o match.

---

# 4. Requisitos derivados

## Normalização — o grosso do trabalho

- **RF-01** — Completar `normalizeAddress()` com os passos 3, 6 e 7
- **RF-02** — Trazer os 4 dicionários como **dado**, não como array no código,
  para permitir curadoria sem deploy
- **RF-03** — Parser de cruzamento: 14 conectores, 7 regras, alimentando
  `first_address` e `second_address`
- **RF-04** — Chave de cruzamento ordenada
- **RF-05** — Preservar a grafia canônica do ISP para exibição, separada da chave
  de busca

## Match

- **RF-06** — Incluir **bairro** como critério
- **RF-07** — Definir o papel da proximidade geográfica, que saiu do
  `findSimilarRecord` na release

## Fontes

- **RF-08** — `cet_code` e CET no filtro (integrar a branch)
- **RF-09** — Cadastrar CBMRJ (id 12) no domínio do banco de dev
- **RF-10** — Definir o escopo do SMS

---

# 5. Perguntas em aberto

**1. O `migracao.py` ainda é o caminho para o ISP?**
Existe `importer-isp.service.ts` na API, com endpoint e template. Estamos
rodando um script manual para algo que virou funcionalidade?

**2. Qual o critério de "nome de via" — igualdade ou similaridade?**
O `findSimilarRecord` novo usa `normalizeAddress` + `compareTwoStrings`.
Confirmar o limiar contra o que o Caio espera.

**3. A proximidade de 100 m foi removida de propósito?**
Ela existia na `main` e sumiu na release. Foi decisão ou efeito colateral da
reescrita?

**4. São dois ou três critérios?**
O slide diz "os DOIS critérios" mas lista três, com asterisco no bairro. E o
bairro hoje não é usado.

**5. Quando a branch do CET entra?**
`feature/implementation-cet-data-import-VDS-879` está parada desde 28/01/2026,
enquanto a release avançou até junho. Ainda é integrável?

**6. O que está em produção?**
A release é de 17/06/2026 e os containers de produção têm imagens de 2–4 meses.
Confirmar se produção roda a release ou algo anterior.

**7. O `accident_code` colidido afeta o match?**
Ver [[Importar a planilha do ISP]] — 1.625 códigos repetidos na carga do ISP
feita pelo script Python. Se o match ou os UPDATEs usarem esse campo, herdam a
colisão.
