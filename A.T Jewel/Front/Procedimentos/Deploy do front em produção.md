# Deploy do front em produção

Manual, sem pipeline. Verificado em 10/08/2026.

> **`AT-AGENTE` é produção.** Não existe homolog — só a máquina do Lucas e este
> servidor. Ver [[Aplicar migração de banco]] para o lado do backend.

---

## Pré-requisito: VPN

A VPN SSL da FortiClient precisa estar conectada, e **só pela interface
gráfica** — a versão 7.4.3.4726 não tem CLI de conexão:

```
C:\Program Files\Fortinet\FortiClient\FortiClient.exe
```

Sem o túnel não existe rota para `10.29.0.0/24` e o SSH falha no TCP, antes de
autenticar. Para conferir se o túnel subiu:

```powershell
Get-NetRoute | Where-Object { $_.DestinationPrefix -like '10.29*' }
```

O VS Code Remote-SSH usa o mesmo `ssh.exe`, o mesmo `~/.ssh/config` e as mesmas
rotas — não tem caminho de rede independente. Se o terminal não conecta, o VS
Code também não vai.

---

## O deploy

```bash
ssh "AT-Front/Back"            # alias no ~/.ssh/config; hostname real é AT-AGENTE
cd ~/at-jewel/at-jewel-front

git status                     # tem que estar limpo — pull sobrescreve edição feita no servidor
git pull origin main
docker restart atjewel_app
docker logs --tail 40 -f atjewel_app
```

Depois, **`Ctrl+Shift+R`** no navegador. Refresh normal serve CSS e fonte do
cache.

**Containers:** `atjewel_app` (front, 80→3000), `atjewel_api` (back, 3000),
`atjewel_postgres` (5432).

---

## A armadilha do cache

Aconteceu em 10/08/2026: o TSX novo apareceu e o CSS continuou o antigo. A tela
de login veio com o HTML novo e as cores velhas.

**Sintoma característico:** classes de valor arbitrário (`text-[#E8D5AB]`)
funcionam e tokens do `@theme` (`bg-nav`, `text-ink`) não.

**Causa:** `/app` é bind mount da pasta do host, mas `/app/node_modules` e
`/app/.next` são **volumes nomeados** — montagem interna ganha da externa. O
`git pull` atualiza o código, o TSX recompila, e a compilação antiga do CSS
sobrevive ao restart.

**Correção:**

```bash
docker exec atjewel_app sh -c 'rm -rf /app/.next/* /app/.next/.[!.]* 2>/dev/null; true'
docker restart atjewel_app
```

`rm -rf /app/.next` inteiro dá "device or resource busy" — o ponto de montagem
não pode sair.

---

## Que ação cada mudança exige

| Mudou | Ação |
|---|---|
| `src/`, `.tsx`, `.css` | nada — o `next dev` recompila sozinho |
| `next.config.ts`, `.env` | `docker restart atjewel_app` |
| `package.json` | `npm install` **dentro** do container (`node_modules` é volume) |
| CSS velho / nada funciona | limpar o `.next` e reiniciar |
| `Dockerfile` | `docker compose up -d --build` |

---

## Por que esse ambiente se comporta assim

Produção roda **`next dev` com Turbopack**, `NODE_ENV: development`, código por
bind mount, atrás de túnel Cloudflare. Não é build de produção.

Consequências no dia a dia:

- **Lento** — compila sob demanda por rota. `/login` levou 4,1s no primeiro
  acesso; com `next build` seria ~20ms
- **Sem otimização** — sem minificação nem tree-shaking; bundles muito maiores
- **Mutável** — qualquer um com SSH edita arquivo direto na pasta. O
  `git status` acusa, mas só se alguém olhar

É dívida real, não provisório. Registrado em
[[2026-08-10]] e em [[2026-08-12]].
