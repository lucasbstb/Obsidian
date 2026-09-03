# ONDE ESTAMOS

> Retomada rápida. Atualizado em **02/09/2026, fim do dia**.
> Detalhe completo em [[2026-09-02]], [[Rotinas diárias]],
> [[Match entre fontes — situação atual e requisitos]].

---

## O essencial em 10 linhas

1. **Trabalhe na `lucasdev`**, que sai da `release/1.0.0` — e a release é o que
   roda em produção. A `main` é a versão de **Fortaleza**, 76/89 commits atrás.
   Login verde = errado; azul "Prefeitura Rio" = certo
2. **Produção** = `da3e19e` (back) / `2da46ef` (front), 17/06/2026 — e o **local
   está idêntico**
3. **Dev está 3 meses atrás** e com back e front em branches diferentes
4. `npm install --legacy-peer-deps` sempre
5. Banco local aponta para o **dev** (`10.29.0.167`), com Redis e Mongo no mesmo host
6. A **produção lê o banco do CET** (`10.39.64.110`) — que roda num Docker
   Desktop de workstation Windows
7. Rede é **unidirecional**: Proxmox → CET funciona, CET → Proxmox não
8. O `migracao.py` está **obsoleto** — a API tem importador de ISP melhor
9. As **ações** de match/não-match já estão implementadas; falta o **critério**
10. Credenciais só em [[Rotinas diárias]], que não vai para o git

---

## Ambiente local

```
back   C:\Users\Lucas Barbosa\Documents\vida-rio-api   lucasdev  ←  release/1.0.0 (da3e19e)
front  C:\Users\Lucas Barbosa\Documents\VIDA-RIO       lucasdev  ←  release/1.0.0 (2da46ef)

npm run dev  nos dois  ·  FortiClient conectado  ·  localhost:3000
```

Login em `/`, não em `/login`. Cookie: `vida-web-rio.token`.

### A branch `lucasdev`

Criada em **02/09/2026**, nos dois repositórios, a partir da `release/1.0.0`.
Existe porque a release **é** produção — mesmo commit, back e front. Nada meu
vai direto para lá; só por PR.

Primeiro commit: `aea2e77` — o `package-lock.json` reescrito pelo
`npm install --legacy-peer-deps`.

> ⚠️ **Esse commit não volta para a release.** É artefato do ambiente local: o
> `typeorm@0.3.20` declara `peerOptional mongodb@^5.8.0` e o projeto usa
> `mongodb@^7.1.0`, o que quebra o install normal com `ERESOLVE`. Ao abrir PR,
> deixar o arquivo de fora.

---

## Estado da migração do ISP no `vidadev` (máquina do CET)

Rodando pelo `migracao.py`, bloco a bloco. Roteiro em
[[Importar a planilha do ISP]].

| Bloco | Resultado | Proporção | |
|---|---|---|---|
| 1 — Sinistros | 38.592 | — | ✅ |
| 2 — Vítimas | 47.760 | 1,24 | ✅ |
| 3 — Vias | 40.539 | 1,05 | ✅ |
| 4 — Veículos | 47.816 | 1,24 | ✅ |
| 5 — UPDATE colunas | **adiado** — bug na linha 683 | | ⚠️ |
| 6 — UPDATE horário | **rodando** (02/09, fim do dia) | | ⏳ |

**Os blocos 1 a 4 fecharam** — as quatro proporções batem com a base
anterior.

### Estado do bloco 6 — retomar por aqui

O teste com o `break` **passou**: o sinistro `00049681` saiu de meia-noite para
`2022-03-11 11:00:00+00`, sem virar `NULL`. Isso confirmou o que faltava — que
`atualizar_horario_registros()` devolve hora de verdade.

O `break` foi comentado e a rodada completa está em execução. **Não leva
`TRUNCATE`**: é `UPDATE`, não `INSERT`, e o `occurred_at::date` o torna
idempotente — o registro de teste é só reescrito com o mesmo valor.

**Quando terminar, rodar:**

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select count(*) filter (where occurred_at::time='00:00') meianoite, count(*) filter (where occurred_at is null) nulo, count(*) total from incident_trafficincident"
```

| Coluna | Esperado |
|---|---|
| `meianoite` | cair para perto de zero |
| `nulo` | **zero** |
| `total` | 38.592 |

> ⚠️ A coluna `nulo` é a que importa. O teste cobriu um sinistro que **tinha**
> hora na planilha. Se `atualizar_horario_registros()` devolver `None` para os
> que **não** têm, o `make_interval(hours => NULL)` anula o `occurred_at` — e aí
> a saída é restaurar do dump, porque `occurred_at::date` sobre `NULL` continua
> `NULL`.

### O bloco 5 fica pendente

A linha 683 passa `incidente`, variável do bloco 4, que deixa de existir quando
se comenta o bloco 4 — `NameError`. Três saídas, em
[[Importar a planilha do ISP]]. Não bloqueia nada; a parte útil dele é a
severidade da lesão, que usa `WHERE name` e nem passa pelo `accident_code`.

**Os blocos 5 e 6 dependem do `accident_code`**, que colide. Medido em
02/09/2026, sobre os 38.592 sinistros carregados:

| | |
|---|---|
| **91** | códigos que colidem (o maior deles em 666 sinistros) |
| **1.716** | sinistros envolvidos — **4,4% da base** |
| 1.625 | linhas excedentes (`1.716 − 91`, e também `38.592 − 36.967`) |

> **Isso libera o bloco 6.** Hoje **100%** da base está à meia-noite; depois dele,
> 95,6% fica com horário certo. E o `occurred_at::date` torna o `UPDATE`
> idempotente — corrigido o `accident_code` depois, basta rodar de novo e os
> 1.716 se acertam. Dump antes, mesmo assim.

O **bloco 5 não roda como está**: a linha 683 referencia `incidente`, variável do
bloco 4, que deixa de existir quando se comenta o bloco 4. Ver
[[Importar a planilha do ISP]].

### Problemas conhecidos dessa carga

- `accident_code` colidido — o script trunca o `controle` no 1º hífen
- **Cor de pele e escolaridade não importadas** — 100% em "Não informado"
- Ambos recuperáveis depois: o `incident_victim.name` guarda o `chave_vitima`

---

## O que travou e por quê

**A planilha do ISP está na máquina do CET e não sai de lá.**

```
CET → internet     ❌
CET → Proxmox      ❌  (testado: .161 e .167, portas 3000 e 8000)
Você → CET         ❌
Você → Proxmox     ✅  (tudo: 22, 80, 3000, 8000)
```

Copiar/colar de texto no RDP está bloqueado. Caminhos ainda não testados:

- [ ] `Test-Path \\tsclient\C` num PowerShell **não elevado** (o erro anterior é
      sintoma de terminal elevado)
- [ ] `Ctrl+C`/`Ctrl+V` do **arquivo** pelo Explorer — canal separado do texto
- [ ] **Pedir a planilha por e-mail** a quem a colocou lá — o único que não
      depende de política de segurança

> Se o arquivo chegar à sua máquina, dá para subir pela tela de Importadores
> (`10.29.0.167:3000` para dev, `.161` para produção) — a API é a ponte até o
> banco.

---

## Próximos passos, em ordem

### 1. Decidir sobre os blocos 5 e 6
- [x] ~~Conferir o bloco 4~~ — 47.816 veículos, razão 1,24 ✅
- [ ] **Tirar dump antes de qualquer coisa**
- [ ] Decidir: rodar 5 e 6 sabendo do `accident_code` colidido, ou não rodar
- [ ] **Ou** abandonar a carga: confirmar com o Yerlon se o `migracao.py` ainda
      faz sentido dado que existe `importer-isp.service.ts`

> **Enquanto o bloco 6 não rodar, todos os sinistros estão à meia-noite.** Isso
> agora atrapalha mais do que antes: o critério de match da COR usa janela de
> ±3h, e numa base assim todo sinistro cai na janela de todos os outros.
> Testar o importador contra essa carga dá resultado sem sentido.

### 2. Tirar a planilha da máquina do CET
Os três caminhos acima.

### 3. Testar o `POST /isp` da API
Com um recorte de ~50 linhas, no ambiente **dev**. Confirma se o mapeamento de
cor de pele entra certo e se o `accident_code` sai sem colisão.
⚠️ A rota é **síncrona**, sem fila — planilha inteira provavelmente estoura o
timeout do nginx.

### 4. Commits pendentes no front
- [ ] Corrigir `scholarityId`/`skinColorId` → `scholarity`/`skin-color` em
      `incidentOptionsService.ts` *(o bug existe na release)*
- [ ] Corrigir `vida-webrio.token` → `vida-web-rio.token` em
      `AnalyzeDuplicity/index.tsx`
- [ ] `.gitignore` do front ignora `*.json` — repositório não é clonável

### 5. Levar ao Caio
- Bairro é critério obrigatório ou opcional? (hoje não é usado por ninguém)
- COR e CBRJ devem convergir para o mesmo critério? (hoje: geografia × nome de via)
- Qual o raio correto? (já foi 100, 1000, hoje 500 m)
- `second_address` volta para a comparação?

### 6. Levar ao Yerlon
- O `migracao.py` ainda é o caminho para o ISP?
- Cadastro manual deixar de checar duplicidade foi decisão?
- Quando a branch do CET entra? (parada desde 28/01)
- Como a COR foi carregada antes? (a tela existe na release)

### 7. Riscos a registrar formalmente
- Banco de **produção** num Docker Desktop de workstation Windows
- `#HOMOLOG DB` no `.env` de dev apontando para produção
- Credencial trivial e idêntica nos três bancos
- Chave da Google Maps hardcoded e versionada
- Geocoding fixado em `Fortaleza, CE` — **ainda na release**
- LGPD: 47 mil vítimas com nome, documento, CNH e data de óbito

---

## O que falta em ISP × COR (resumo)

**ISP:** nada. É a fonte primária.

**COR:** quatro itens —
1. Dedup interna ("deixar o último") — não existe; o CBMRJ tem, é copiar
2. Critério por nome de via — hoje usa geografia 500 m
3. Critério por bairro — não usa
4. Normalização completa — 5 de 7 passos, dicionário com 8 de ~40 entradas

As **ações** de match e não-match **já estão prontas** e comprovadas nos dados.
