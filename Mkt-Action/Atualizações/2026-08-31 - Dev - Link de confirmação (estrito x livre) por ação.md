# 2026-08-31 — Link de confirmação (estrito × livre) por ação

**Tipo:** desenvolvimento (feature) · **Branch:** `feat/link-confirmacao-por-acao` · **Commit:** `7ad497c` (local, **não** pushed) · **Migration:** `0277`

## O problema
O link do **cabeçalho** ("Compartilhar link") sempre pedia **CPF/telefone** (`/a/`),
enquanto o link do **líder** (`/i/`) sempre abria o fluxo livre ("Confirmar e deixar
contato / só confirmar"). Não dava pra escolher: tem ação que **exige** identificação
(fiscalização) e tem ação que quer **adesão livre**.

## A solução — um switch por ação
Toda ação já tinha **dois links públicos**:

| Link | Rota | Comportamento |
|---|---|---|
| **Estrito** | `/a/{public_code}` | pede **CPF ou telefone** (colaborador já cadastrado) |
| **Livre** | `/e/{event_code}` · `/i/{invite_code}` | confirma **sem exigir contato** (opcional) |

Agora o gestor **escolhe por ação** qual deles o link do colaborador usa, por um
**checkbox** no passo **Colaboradores** (e na aba **Importar** da ação):

> ☑️ **Exigir CPF ou telefone na confirmação** — padrão **marcado (estrito)**.

- **Marcado (estrito):** cabeçalho compartilha o `/a/`; o link do líder `/i/`
  **redireciona pro `/a/`** (mesma tela de CPF/telefone).
- **Desmarcado (livre):** cabeçalho compartilha o `/e/`; o link do líder volta ao
  "Confirmar e deixar contato / só confirmar".

O checkbox salva na hora (`PATCH`) na coluna `actions.confirm_requires_id`.

## Como funciona (técnico)
- **Migration `0277`:** `actions.confirm_requires_id boolean NOT NULL DEFAULT true`.
- **Cabeçalho** (`layout.tsx` + `action-header.tsx` + `copy-link-button.tsx`): monta a
  URL conforme o modo e adapta os textos (CPF/telefone × livre), inclusive na mensagem
  do WhatsApp.
- **Checkbox** (`caderno-builder.tsx`): lê o valor atual da ação e faz `PATCH` ao alternar.
- **Link do líder** (`i/[code]/page.tsx` + `api/i/[code]/route.ts`): a API devolve
  `requires_id` + `public_code`; a página redireciona pro `/a/` quando estrito.
- **Bônus:** corrigido o texto do `CopyLinkButton` no **QR** e no **convite do captador**
  (são links livres, mas mostravam "CPF/telefone" por engano).

### Arquivos alterados
`0277_link_do_colaborador_por_acao.sql` · `api/actions/route.ts` ·
`api/actions/[id]/route.ts` · `acoes/[id]/layout.tsx` · `action-header.tsx` ·
`copy-link-button.tsx` · `caderno-builder.tsx` · `acoes/[id]/qr/page.tsx` ·
`acoes/[id]/captadores/page.tsx` · `i/[code]/page.tsx` · `api/i/[code]/route.ts`

## Como testar (local)
1. Abre uma ação → aba **Colaboradores** → o checkbox aparece no topo (marcado = estrito).
2. Com **estrito**: clica "Compartilhar link" no cabeçalho → link `/a/...` (pede CPF/tel).
   O link do líder `/i/...` também cai na tela de CPF/telefone.
3. **Desmarca** → tudo vira livre (link `/e/...`, líder volta ao um-clique).

⚠️ A tela `/a/` só aceita **colaborador já cadastrado** na ação (é o ponto do estrito).
Pra testar de verdade, adiciona um colaborador e usa o telefone dele.

## Decisão tomada (a confirmar com o cliente)
O **QR anônimo do evento (`/e/`)** ficou **de fora** do switch de propósito: é headcount
público **sem fiscalização** (RF-LID-28), e como o padrão é estrito, redirecioná-lo
quebraria o QR de toda ação. **Só o link do líder (`/i/`)** segue o estrito.
→ Se o Germano quiser que o **QR do evento também** exija CPF no modo estrito, é o mesmo
padrão — fácil de adicionar.

## Estado / próximos passos
- [x] Implementado e verificado local (`tsc` = 0 erros, eslint limpo, migration aplicada).
- [x] Commitado local no branch `feat/link-confirmacao-por-acao` (`7ad497c`), **sem push**.
- [ ] Validar no navegador com um colaborador cadastrado.
- [ ] Decidir com o cliente se o QR do evento (`/e/`) também segue o estrito.
- [ ] Push pro GitLab (`git push origin feat/link-confirmacao-por-acao`) quando aprovar.
- [ ] Aplicar a `0277` no ambiente ao subir (homolog/prod).
