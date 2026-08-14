# Requisitos — Reunião (Suíça): múltiplas áreas por liderança, geografia e dobradinhas

> **Fonte:** ata/resumo da reunião (cliente **Suíça** — Fortaleza + Crato). Action items
> atribuídos a **Thiago Parente**.
> **Estado conferido no código** (conferência item a item). ⚠️ **Duas bases hoje:** a
> `main` (produção, deployada, com cripto + migrations até ~0242) e a `lucasdev` (protótipo
> de layout, migrations até 0218). Onde as duas divergem, está apontado — a **`main` é a
> verdade** para comportamento/schema.
> **Legenda:** ✅ Pronto · 🟡 Parcial (falta parte) · 🔴 A fazer · 🟣 Decisão aberta · ⚪ Não-código.

**Placar:** 2 prontos · 7 parciais · 3 a fazer · 3 decisões abertas · 4 diretrizes/não-código.

> 🎯 **O item central da reunião** é o **RF64 — múltiplas áreas por liderança**. Ele destrava
> quase todo o resto (mapa por múltiplas zonas, quantidade por área, rateio). Hoje um líder
> tem **uma** zona e **um** bairro de atuação — o modelo precisa virar N áreas.

---

## Tema A — Cadastro geográfico da liderança

| ID | Requisito (reunião) | Status | Onde está / o que falta |
|----|--------------------|--------|--------------------------|
| **RF64** | **Múltiplas zonas/bairros/cidades por liderança** — refletir atuação real (líder atua em vários lugares). | 🔴 A fazer | Hoje é **campo único**: `org_leaders.zona_atuacao` + `bairro_atuacao` (migration `0196:145-147`, "onde a liderança ATUA / RF41") e o par de votação (`0184:31-40`, "onde o líder VOTA"). **Não há** tabela de junção (`leader_areas`). Precisa de tabela `leader_areas(leader_id, tipo['zona'|'bairro'|'cidade'], valor, qtd_colaboradores)` + UI de múltipla seleção. **É a base dos demais.** |
| **RF65** | **Quantidade de colaboradores por área** (ex.: 20 na zona X, 15 no bairro Y). | 🟡 Parcial | Hoje a contagem por área é **derivada** do cadastro 1-a-1 (`contacts`/`workers`) e agregada no RPC de mapa (`0097:96-112`). **Não há campo de quantidade digitada por área.** Casa com o RF64 (a quantidade viraria coluna da `leader_areas`). |
| **RF66** | **Apelido/alias da liderança** ("Marcão das Ostras"). | ✅ Pronto | `org_leaders.apelido` (`0184:40`), já editável em `leader-edit-modal.tsx`. *(Nada a fazer — validar na demo.)* |
| **RF67** | **Macro-regiões e "Regional"** — agrupar várias cidades; "Regional" como subdivisão (só existe em Fortaleza). | 🟡 Parcial | Existe o **escopo** da org: `organizations.primary_uf`/`scope_type` + tabela `organization_regions` (`0095:7-13`) e a região travada (`0141`). **Não existe** macro-região hierárquica agrupando cidades, nem "Regional" como subdivisão intra-cidade. Precisa cadastro de regiões/regionais + vínculo. *Ver decisão #1 e action item "consultar Claudio".* |

## Tema B — Contabilização de votos e colaboradores

| ID       | Requisito                                                                                                                    | Status                                  | Onde está / o que falta                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| -------- | ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RF68** | **Promessas não-cumulativas** — líder promete 10 mil; sublíderes não somam duplicado; mostrar quanto já veio × quanto falta. | 🟡 Parcial (✅ em boa parte na **main**) | **Na `main`:** a família `0238/0239/0242` fecha a **contenção hierárquica** (pai não soma com filho) e separa `esperados_lideres` × `esperados_comite` (evitou o bug dos 110.000 no dashboard). **Na `lucasdev`:** só a contenção limitada por líder (`get_action_estimativa`, `0215:58` — usa `COALESCE(declarado, nomeados)`, "nunca soma os dois"), **sem** a regra pai→sublíder. Ou seja: **já resolvido na produção**; falta só portar a leitura/dashboard. |
| **RF69** | **Diferenciar colaborador identificado × anônimo × equipe.**                                                                 | 🟡 Parcial                              | Identificados = `contacts` com dados. Anônimos (promessa sem nome) = `org_leaders.colaboradores_estimados` (`0191:20`, RPC `get_org_leader_estimates:42`) e o "+N" de `contacts.expectativa` (`0217:10`). Equipe = aba Equipe (papel/função/coordenador, `0196`/`0218`). **Falta:** rotular claramente o **"anônimo"** na interface (hoje o conceito existe, mas não é nomeado pro gestor — action item da reunião).                                             |
| **RF70** | **Presença/comparecimento por check-in + QR** (identificado e anônimo).                                                      | ✅ Pronto                                | `event_checkins` + telas públicas `/a` (CPF/tel), `/i` (líder), `/e` (QR de parede). Presença anônima contabilizada; RF47/RF63 (contato do QR vira participante) já feitos. *(Validar na demo.)*                                                                                                                                                                                                                                                                 |
| **RF71** | **Colaborador em múltiplas áreas com rateio proporcional/ponderado.**                                                        | 🟣 Decisão aberta                       | `contacts` tem zona/bairro/cidade **únicos**. abrir possibilidade de cadastrar mais de uma ou varias. *Ver decisão #2 — risco de complexidade no cadastro.*                                                                                                                                                                                                                                                                                                      |

## Tema C — Visualização geográfica

| ID | Requisito | Status | Onde está / o que falta |
|----|-----------|--------|--------------------------|
| **RF72** | **Mapa por bairro (Fortaleza) e por cidade (outras regiões)** — em vez de só zona. | 🟡 Parcial | O **choropleth por bairro/município já existe** no dashboard (`coverage-choropleth-map.tsx:367-625`; RPC `0097` aceita `p_level='bairro'`; `api/dashboard/bairros`). Votação histórica TSE ainda é por **zona** (`0047`, `inline-map.tsx`). **Falta:** priorizar/unificar a visão bairro-ou-cidade na interface principal (hoje convive com a de zona). |
| **RF73** | **Ver múltiplas zonas/bairros de UM líder num mapa integrado.** | 🔴 A fazer | Depende do **RF64** (líder só tem uma área hoje, então não há o que sobrepor). Sai de graça quando a `leader_areas` existir. |
| **RF74** | **Dobradinhas** — candidato federal + estadual atuando juntos, com zonas/bairros, votos e colaboradores. | 🔴 A fazer | **Não existe** conceito de dobradinha/par/coligação. O que há é comparação **visual** de 2 candidatos (Voronoi, `inline-map.tsx:9`; tool `compare_candidates_map`) — análise histórica ad-hoc, não cadastro. Precisa entidade `dobradinha` (2 candidatos + áreas + votos/colab). *Ver decisão #3.* |
| **RF75** | **Cruzamentos + relatórios geográficos com exportação PDF/gráficos** pra apresentação. | 🟡 Parcial | Cruzamentos por macro-região/cidade/tipologia existem (agente/analytics). **Exportação só `.xlsx`** (`api/caderno/export`, `api/export`); **PDF não existe** (sem `jspdf`/`puppeteer`). Só há download de imagem do mapa. Falta relatório PDF montado com gráficos. |

## Tema D — Cadastro, importação e metas

| ID       | Requisito                                                                                                         | Status     | Onde está / o que falta                                                                                                                                                                                                                              |
| -------- | ----------------------------------------------------------------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RF76** | **Importação combina lista completa + estimativa anônima** na totalização (quando não há lista, só a quantidade). | 🟡 Parcial | Import de listas existe (`api/import`, `api/contacts/import`). Estimativa anônima por líder existe **separada** (`colaboradores_estimados`). **Falta:** somar identificados + anônimos num total único de mobilização por líder/área.                |
| **RF77** | **Segmentação escolhível (zona/bairro/cidade) com quantidades no cadastro.**                                      | 🟡 Parcial | É a ponta de UI do **RF64/RF65** — o usuário escolhe o nível e informa quantidade. Depende da `leader_areas`.                                                                                                                                        |
| **RF78** | **Dashboard meta × alcançado × falta** (de votos da campanha).                                                    | 🔴 A fazer | **Não existe** meta de votos configurável (grep `meta_votos`/`vote_goal` = 0). O dashboard mostra **histórico TSE** (`candidate-summary.ts:18`, "votos que faltaram pro 1º em 2022"), não meta da campanha atual. Precisa campo de meta + progresso. |
|          |                                                                                                                   |            |                                                                                                                                                                                                                                                      |

## Tema CHAT

Toda e qualquer "figura" como mapa, foto, imagem que tenha gráficos e afins sejam exportáveis em jpeg. 
## Tema E — Diretrizes (não-código)

| Item | Status | Nota |
|------|--------|------|
| **Treinar a Suíça** no uso de filtros (zona/bairro/cidade) e leitura dos dados | ⚪ Não-código | Evita erro de entrada e maximiza valor. Responsável: Thiago. |
| **Comunicar** que "Regional" só existe em Fortaleza | ⚪ Não-código | Outras cidades não têm essa subdivisão — alinhar expectativa. |
| **Recomendar apoio** (pessoa dedicada) pro cadastro correto das áreas | ⚪ Não-código | Cadastro de múltiplas áreas dá trabalho operacional. |
| **Consultar Claudio** (especialista) pra modelar região×bairro×zona×macro-região | ⚪ Não-código | Decisão técnica de modelagem antes de codar RF64/RF67. |

---

## Próximas ações (mapeadas dos action items → RF)

- [ ] **RF64 + RF65 + RF77** — modelo de **múltiplas áreas por liderança** com quantidade por localidade (`leader_areas`) + UI. *(action item "Ajustar cadastro de múltiplas zonas/bairros/cidades por liderança".)*
- [ ] **RF69** — rotular **colaborador anônimo** claramente na interface; garantir que promessas sem identificação contam e aparecem. *(action item "revisar anônimos e votos prometidos".)* — parte da regra já está na **main** (RF68).
- [ ] **RF74 + RF72** — **dobradinhas** (federal+estadual) + mapa por bairro/zona. *(action items "integrar mapa por bairro/zona" e "dobradinhas".)*
- [ ] **RF75** — melhorar **relatórios/PDF** com gráficos formatados pra apresentação.
- [ ] **RF78** — **meta × alcançado × falta** de votos no dashboard.
- [ ] Portar pra `main` o que estiver na `lucasdev` e vice-versa (RF68 já está na main).

## Decisões abertas (antes de codar)

1. **🟣 Macro-região / Regional (RF67):** como modelar? "Regional" só vale pra Fortaleza — as outras cidades param no nível cidade/UF. Precisa da definição do **Claudio** antes de criar tabela. *Qual a hierarquia oficial: UF → macro-região → cidade → (regional só Fortaleza) → bairro → zona?*
2. **🟣 Rateio em múltiplas áreas (RF71):** peso percentual (ex.: 60% zona A / 40% zona B) **ou** contagem inteira por área? A reunião pede "regras de rateio simples pra não complicar o cadastro" — decidir o mais simples que atende.
3. **🟣 Dobradinha (RF74):** é um **cadastro** (entidade com 2 candidatos + áreas + votos/colab) ou basta a **comparação visual** que já existe? Definir o escopo antes.

## Já pronto / apenas validar na demo

- **RF66** apelido da liderança ✅
- **RF70** presença por check-in + QR (identificado e anônimo) ✅
- **RF68** contenção de votos prometidos — **já na `main`** (`0238/0239/0242`) ✅
- Choropleth por bairro/município no dashboard (base do RF72) ✅

---

*Documento gerado a partir da ata da reunião (Suíça), conferido item a item contra o código
(`main` para comportamento/schema, `lucasdev` para layout). Editável — ajuste antes de aprovar
a implementação. O RF64 é o pré-requisito da maior parte do resto.*
