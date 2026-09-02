# Arquitetura e mapa do sistema

> Documento vivo — atualizar conforme o entendimento do sistema evoluir.
> Relacionados: [[Rotinas diárias]], [[Acessar os bancos]],
> [[Importar a planilha do ISP]], [[2026-09-01]].

Levantado em 01–02/09/2026, ao assumir o projeto do dev anterior (Yerlon).
Vale para `vida-rio-api` e `VIDA-RIO`.

---

## O que o sistema faz

```
Entrada  →  Qualificação  →  Análise  →  Saída
─────────   ─────────────    ────────    ──────
manual         revisão        mapa       CSV simplificado
SAMU (xlsx)    duplicidade    gráficos   CSV completo (zip)
PRF (xlsx)     imprecisos     painéis    pacote DETRAN
CBMERJ (py)                   gerência   relatórios anuais (PDF)
ISP (py)
```

## Módulos do backend

| Módulo | O que faz |
|---|---|
| `incident` | núcleo — ~70% do código. Registro, mapa, gráficos, importadores, revisão, duplicidades, imprecisos, exportações |
| `management` | metas anuais, janelas de data e painéis no padrão OMS (capacete, cinto, álcool, celular, velocidade) |
| `education` | ações de educação de trânsito. Existe no back, mas **o menu está comentado no front** |
| `transit-service` | cadastro de agentes e ambulâncias |
| `auth` / `user` | login JWT (9h), cadastro, recuperação de senha por e-mail |

## O banco

**PostgreSQL 17 + PostGIS 3.5**, com `synchronize: false` e **sem migrations no
repositório**. O schema é gerenciado fora da aplicação.

> Os nomes de tabela (`auth_user`, `incident_trafficincident`,
> `education_trafficeducation`, colunas com duplo sufixo `traffic_incident_id_id`)
> são assinatura do **Django ORM**. O banco nasceu de uma aplicação Django e o
> NestJS foi construído por cima dele. É a origem dos scripts Python — eles são
> legado daquela era e nunca foram migrados.

### Modelo central

```
incident_trafficincident (sinistro)
├── FK ×11 → tabelas de domínio (natureza, gravidade, iluminação, jurisdição,
│            local, controle de tráfego, tipo de cruzamento, superfície,
│            horário, elemento afetado, fonte primária)
├── 1:N incident_trafficlane   (vias)     → 12 FKs de domínio
├── 1:N incident_vehicle       (veículos) → 4 FKs de domínio
├── 1:N incident_victim        (vítimas)  → 13 FKs de domínio
│         └── 1:N incident_incidentvictiminjury → lesões
├── 1:N incident_image
├── 1:N incident_trafficincidentagent    → transit_services_agent
└── 1:N incident_trafficincidentvehicles → transit_services_ambulance
```

São **97 entidades** mapeadas. O padrão dominante é tabela de domínio para tudo
(~45 tabelas de lookup), todas com `eager: true` — carregar um sinistro traz a
árvore inteira.

### PostGIS

- `incident_trafficincident.geom` — `geometry(Point, 4326)`
- `polygon_boundary.geometry` + `properties` **jsonb** — bairros (tipo 2),
  logradouros (tipo 1), regionais (tipo 3), carregados de shapefiles
- Filtros usam `ST_Intersects(geom, ST_GeomFromGeoJSON(:polygon))`, inclusive
  para polígono desenhado à mão no mapa

> `polygon_boundary` tem **44.534 linhas** e **nunca pode ser truncada** — os
> filtros geográficos do mapa dependem dela.

### Flags de qualidade do dado

`has_archived`, `was_removed`, `is_duplicated`, `was_duplicity_resolved`,
`location_changed`, `location_verified`, `location_type`, `original_address`.

Toda consulta de análise filtra `hasArchived = false AND wasRemoved = false`.
Registros problemáticos ficam no banco, apenas fora das estatísticas.

### Autorização

```
auth_user ──< auth_user_profile >── auth_profile ──< auth_profile_permission >── auth_permission
                                                                                     └─ tag
```

A `tag` (`view_sinister_map`, `add_sinister`, `import_sinister`,
`resolve_duplicity_sinister`, `view_management`…) é resolvida no login,
embarcada no JWT e verificada pelo `RolesGuard`. O front espelha as mesmas tags
no `middleware.ts` e no `nav-options.ts`.

> Se o usuário logar e o menu aparecer vazio, o problema é perfil sem permissão
> em `auth_user_profile`, não o front.

---

## As fontes do Rio — o que existe na `release/1.0.0`

Levantado em 02/09/2026, na branch que roda em produção.

| Fonte | Serviço | Rota | Fila | Coluna | Filtro |
|---|---|---|---|---|---|
| **ISP** | `importer-isp.service.ts` | `POST /isp` | ❌ síncrono | `isp_code` | ✅ |
| **COR** | ⚠️ `importer-cet.service.ts` | `POST /cor` | ✅ | `cor_code` | ✅ |
| **CBRJ** | `importer-cbmrj.service.ts` | `POST /cbrj` | ✅ | `cbrj_code` | ✅ |
| **CET** | ❌ não existe | ❌ | ❌ | ❌ | ❌ |

### ⚠️ Armadilha de nomenclatura

```
importer-cet.service.ts   →  export class ImporterCorService   ← é a COR!
importer-cbmrj.service.ts →  export class ImporterCbmrjService
importer-isp.service.ts   →  export class ImporterIspService
```

O arquivo chamado `importer-cet` contém o importador da **COR**. Foi criado para
a CET e reaproveitado sem renomear.

> Quem procurar o importador da COR pelo nome do arquivo não acha. Quem abrir o
> `importer-cet` achando que é CET vai mexer na COR.

E a rota é `/cbrj` enquanto a classe é `Cbmrj` e o domínio do banco usa CBMRJ
(id 12). Não quebra nada, mas atrapalha a busca.

### O que falta

**A CET não existe nesta branch.** Está em
`feature/implementation-cet-data-import-VDS-879`, parada desde **28/01/2026**,
enquanto a release seguiu até junho. É a **etapa 4** do fluxo dos slides.

**O ISP não usa fila.** COR e CBRJ importam de forma assíncrona, com processador
dedicado; o ISP processa na própria requisição. Com planilha de 14 MB isso pode
estourar timeout — verificar se é intencional.

## As seis fontes de dados

O sinistro guarda um código por origem. **Só duas são importadas pela API:**

| Código | Fonte | Quem preenche |
|---|---|---|
| `samu_code` | SAMU | ✅ `importer-samu.service.ts` |
| `prf_code` | Polícia Rodoviária Federal | ✅ `importer-prf.service.ts` |
| `amc_code` | Autarquia Municipal de Trânsito | ❌ ninguém |
| `pre_code` | Polícia Rodoviária Estadual | ❌ ninguém |
| `pefoce_code` | Perícia Forense | ❌ ninguém |
| `sim_code` | Sist. de Informações sobre Mortalidade (DATASUS) | ❌ ninguém |

Os quatro últimos aparecem **só na exportação** (`export-file.service.ts`).
Nenhuma linha do projeto grava neles — quem grava é o **pipeline Python**, fora
do repositório. Ver [[Importar a planilha do ISP]].

## A herança de Fortaleza

O sistema **nasceu em Fortaleza** e foi replicado para o Rio. As marcas:

- `pefoce_code` = **Perícia Forense do Estado do Ceará**
- `amc_code` = Autarquia Municipal de Trânsito (Fortaleza)
- `get-coordinates-from-google.ts` tem `', Fortaleza, CE, Brasil'` **fixo no
  código**

> ⚠️ **Risco vivo:** qualquer endereço geocodificado pela API cai em Fortaleza.
> Não afeta a importação do ISP (a planilha já traz `geom` pronto em EWKB), mas
> afeta o cadastro manual de sinistro pela tela.

As fontes do Rio são outras: **CBMERJ** (Corpo de Bombeiros) e **ISP**
(Instituto de Segurança Pública).

### A identidade visual também é de Fortaleza

Confirmado ao subir o front em 02/09/2026 — a tela de login exibe **"Fortaleza
Prefeitura"** no topo e **AMC / Prefeitura de Fortaleza / Transitar** no rodapé.

```
public/logos/
├── amc-prefeitura-transitar.svg    AMC + Prefeitura de Fortaleza + Transitar
├── prefeitura.svg                  Prefeitura de Fortaleza
└── vida-*.svg                      logo da plataforma
```

Usados em `FormLogin`, `FormRecoverPassword`, `FormUserChange`,
`FormUserRegister` e `Navbar` — todas as telas de autenticação e o topo do
sistema.

Os arquivos estão versionados, então **o ambiente publicado mostra o mesmo**.
Substituir pela identidade do CET-Rio é troca de asset + referência nos cinco
componentes.

---

## Frontend

Next.js 14.1.3 App Router, Server Components e Server Actions. MUI,
`react-map-gl` + `mapbox-gl` + **deck.gl**, recharts/apexcharts,
react-hook-form + zod. Token JWT em cookie `vida-web.token`.

**Rotas ativas:** `/sinistro/{mapa, painel, registros, revisao, importador,
duplicidades, imprecisos, detalhes/[id]}`, `/gerencia`, `/perfil`.

**Comentadas no menu:** Engenharia, Fiscalização, Educação, Boletim. As pastas e
páginas existem — funcionalidade planejada ou pausada.

---

## Pontos de atenção acumulados

Levantados na análise inicial, todos ainda em aberto:

1. **Chave da Google Maps API hardcoded** em `get-coordinates-from-google.ts:29`,
   versionada no git. Precisa sair para variável de ambiente e ser rotacionada
2. **`POST /api/user/signup` é `@Public()`** — qualquer um cria conta, e recebe
   o perfil `id: 1` automaticamente
3. **Sem migrations** — o schema só existe no servidor
4. **Redis e MongoDB** no `.env` e no `package.json`, com **zero uso no código**
5. **Geocoding fixado em Fortaleza** (acima)
6. **IPs fixos no `nginx.conf`** (`10.29.0.161`) — provavelmente é a máquina de
   **produção** (`Vida-rio`), não a de dev (`.167`). O arquivo versionado é a
   config de produção, e usá-lo no dev aponta para o host errado
7. **Credencial trivial e idêntica** nos três bancos, escutando em `0.0.0.0:5432`
8. **LGPD** — `incident_victim` guarda nome, documento, CNH, cartão do SUS, data
   de óbito e procedimentos clínicos de dezenas de milhares de pessoas reais,
   num ambiente de desenvolvimento
9. **`.gitignore` do front ignora `*.json`** — repositório não clonável
10. **`estrutura.txt` de 10 MB** versionado no front, sem uso
