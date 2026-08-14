# Requisitos — Reunião de 06/08/2026 (estoque + interface pré-lançamento)

> **Fonte:** reunião Germano [cliente] × Thiago [CEO] × Lucas[Dev].
> **Estado conferido no código** (`lucasdev` v0.8.0, migrations até 0217).
> **Legenda:** ✅ Pronto · 🟡 Parcial (falta UI/parte) · 🔴 A fazer · ⚪ Infra/diretriz.

**Placar:** ✅ **os 6 itens a fazer foram IMPLEMENTADOS em 06/08** (na `lucasdev`, commit `bc93895` + push) · 5 já prontos/validados · 4 pontos a confirmar com o Germano.

> ⚠️ O doc antigo `docs/REQUISITOS-REUNIAO-LIDERANCAS-2A-RODADA.md` está **desatualizado**:
> marca RF48/RF49 (estoque) como 🔴 "a construir", mas isso **já foi feito** (commit `b27cbc6`
> + migrations 0215–0217). Atualizar o status lá depois.

---

## Tema A — Interface do painel e tabelas

| ID       | Requisito (reunião)                                                                         | Status     | Onde está / o que falta                                                                                                                                                                                                                                          |
| -------- | ------------------------------------------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RF60** | Remover a **saudação** ("Boa tarde") do topo do painel — layout mais limpo.                 | ✅ Feito 06/08 | Renderizada em `dashboard-hero.tsx:119` (`<p>{greeting}</p>`), alimentada por `dashboard/page.tsx:234`. É ~1 linha. ⚠️ o texto é override-ável pelo cliente (`welcome_title`); remover a `<p>` do hero cobre os dois casos.                                      |
| **RF62** | **Nomes de coluna por extenso + quantidades** na tabela de colaboradores (está comprimida). | ✅ Feito 06/08 | Cabeçalhos em `contatos/page.tsx:777-784`; abreviação **"Zona/Seç."** persiste (781). ⚠️ o **"CO"**(Avatar) citado na reunião **não existe mais no código** — coordenador já aparece como badge "Coordenador" por extenso (843-845). *Ver ponto a confirmar #3.* |

## Tema B — Aba Equipe / Comitê

| ID            | Requisito                                                                                                                                   | Status     | Onde está / o que falta                                                                                                                                                                                                                                                                                             |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RF51**      | Estrutura da aba **Equipe** (apoio: segurança, financeiro, coordenadores; pessoa pode ser equipe **e** colaborador). *Validado na reunião.* | ✅ Pronto   | Aba Equipe (`contatos/page.tsx:100/370`), membro = função **OU** coordenador (`api/contacts/route.ts:72`), agrupada por área. E posso cadastrar um Coordenador e ele ter pessoas que estão diretamente no grupo dele de trabalho. E varios outras coordenadores com seus grupos(mkt, financeiro, designer, e etc..) |
| **RF61**      | **Botão de sincronização** na Equipe — atualizar os membros sem recarregar a página (base >500).                                            | ✅ Feito 06/08    | Não existe botão de refresh; a lista carrega no `useEffect` (`contatos/page.tsx:222`) via `load()`. Esforço baixo — a função já existe, falta o botão. Mas se já existe, precisa ter?                                                                                                                               |
| **RF50/RF52** | **Filtrar por função/área** na lista de colaboradores (segurança, logística…).                                                              | ✅ Feito 06/08 | Filtrar por **coordenador já funciona** (filtro **Papel**, `page.tsx:738-748`). Falta o **chip de Função** na UI (o backend já aceita `?funcao=`, `route.ts:73`). Múltiplas funções (`funcoes[]`) ainda não filtráveis.                                                                                             |
| **RF43a**     | **Hierarquia líder/sublíder** clara no organograma. *Reforçado na reunião.*                                                                 | ✅ Pronto   | `leader-org-chart.tsx` (árvore por `parent_leader_id`, indentação, badge "principal", rollup da rede). Máx. 2 níveis.                                                                                                                                                                                               |

## Tema C — Estoque (item central da reunião)

| ID | Requisito | Status | Onde está / o que falta |
|----|-----------|--------|--------------------------|
| **RF48/RF49** | Ao registrar/encerrar a ação, informar a **quantidade usada** e o sistema **deduzir do estoque** — sem duplicar dado. | ✅ Pronto (funcional) | Captura no encerramento: IA extrai da observação, **gestor confirma a quantidade** (`close-action-prompt.tsx`) → grava em `action_materials`. Saldo = **planilha − consumo** (`get_stock_balance`, migrations 0174/0216), exibido no `stock-card.tsx`. Dimensionamento: fator de quebra `fator_presenca` (0.5) + `por_pessoa` + material sugerido (0215/0216). |

> ⚠️ **Ressalva de arquitetura (ponto a confirmar #1):** a baixa é **saldo-calculado**
> (`total − consumo`), **não** uma baixa destrutiva que reescreve o número da planilha. Foi
> decisão de projeto (header da migration 0174) pra não quebrar no próximo upload de
> planilha. **Funciona como a reunião pediu**; só confirmar se o cliente espera *ver o
> número do estoque diminuir* na tela.

## Tema D — Presença por QR Code

| ID            | Requisito                                                                                          | Status     | Onde está / o que falta                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ------------- | -------------------------------------------------------------------------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RF47**      | Trocar o **checkbox** de autorização por um **botão de confirmação** que salve o contato digitado. | ✅ Feito 06/08 | Hoje: gaveta opcional com **checkbox de consentimento LGPD** (`e/[code]/page.tsx:172`); sem o check, o backend descarta o contato (`api/e/[code]/route.ts:91`, constraint `event_checkins_consent_required`). Pedido = botão dedicado que garanta o salvamento. *Não é só UI — mexe na semântica do consentimento. Ver ponto a confirmar #2.*<br>vamos deixar do jeito que está.. Mas vamos melhorar o layout para ser mobile-first.. hoje ele abre pequeno demais.. letras muidas..  |
| **RF33/RF46** | Evitar **múltiplas confirmações pelo mesmo aparelho**.                                             | ✅ Pronto   | Dedup por `device_id` (cookie) em `api/e/[code]/route.ts`: aparelho repetido retorna `already:true` sem criar linha. Sinais de fraude via `get_checkin_signals` (0194). *Ressalva: aparelhos diferentes / limpar cookie só é sinalizado, não bloqueado. Ver ponto a confirmar #4.*                                                                                                                                                                                                    |
OBS: Quando coloco que desejo informar o contato na hora da presença.. não vejo o contato.. gostaria que quando fosse informado ele contabilizasse como presença na campanha e entrasse como um cadastro do lider: direto..
**→ ✅ RF63 FEITO (06/08):** o contato do QR agora entra no caderno como **Direto** E vira **participante PRESENTE da ação** (worker no captador Direto, status `started`, preso ao ponto do mapa). Dedup por telefone.
## Tema E — Infra e cronograma (não é código)

| Item | Status | Nota |
|------|--------|------|
| Migração pra **AWS** (sai da degustação → ambiente real da campanha) | ⚪ DevOps | Responsável Thiago. |
| Fechar a **versão atual esta semana** (campanha em ~10 dias) | ⚪ Diretriz | Possível expansão futura pra campanha de **governo**. |

---

## Próximas ações desta reunião (responsável: Lucas) — ✅ TODAS FEITAS (06/08)

- [x] **RF60** — saudação removida (mantém welcome_title custom se houver)
- [x] **RF61** — botão "Atualizar" no Comitê (recarrega sem F5)
- [x] **RF50/52** — chip de filtro por Função (backend casa `funcao` OU `funcoes[]`)
- [x] **RF47** — QR mobile-first: 2 botões (só presença × +contato) + máscara no WhatsApp
- [x] **RF62** — lateral sem avatar (280px) + tabela limpa (esconde CPF/Bairro/Zona) + edição pela modal de cadastro
- [x] **RF63** — contato do QR vira cadastro Direto no caderno **+** participante PRESENTE da ação

> Tudo em `lucasdev` (commits `bc93895` feat + `6a6b626` docs, push feito). Port pra `main`
> (cripto + refactor do Thiago) em `docs/HANDOFF-THIAGO-2026-08-06.md`.

## Pontos a confirmar com o Germano (antes de codar)

1. **Estoque:** o modelo é saldo-calculado (`total − consumo`), não baixa destrutiva. O
   cliente aceita, ou espera ver o *total da planilha diminuir* na tela?
2. **QR:** o consentimento LGPD passa a ser **implícito no clique** do botão de confirmação?
   (afeta backend + a constraint de consentimento.)
3. **Tabela:** o "CO" que incomodava **não existe mais** no código (já é "Coordenador" por
   extenso). Qual coluna ainda incomoda além de "Zona/Seç."?
4. **QR:** "resolver múltiplas confirmações" = o dedup por aparelho basta, ou querem
   **bloqueio mais forte** (que só dá pra *sinalizar*, não impedir — mesma pessoa em outro
   celular)?

## Já pronto / apenas validado na reunião (nada a fazer)

- Estrutura da aba Equipe (**RF51**) ✅
- Hierarquia líder/sublíder no organograma (**RF43a**) ✅
- Baixa de estoque pela ação (**RF48/49**) ✅ *(ver ressalva #1)*
- Dedup de presença por aparelho no QR (**RF33/46**) ✅
- Filtrar por coordenador (filtro Papel) ✅

---

*Documento gerado a partir do resumo/ata da reunião de 06/08/2026, conferido item a item
contra o código em `lucasdev` (v0.8.0). Editável — ajuste o que precisar antes de aprovar
a implementação.*
