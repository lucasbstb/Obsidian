# Acessar os bancos

> Procedimento vivo — atualizar quando a topologia ou os acessos mudarem.
> Relacionados: [[Rotinas diárias]], [[Backup e restore do banco]],
> [[Arquitetura e mapa do sistema]], [[2026-09-02]].

Estabelecido em 02/09/2026, ao mapear a infraestrutura herdada.

---

## Os bancos

Todos se chamam parecido e todos usam a **mesma credencial** — valor em [[Rotinas diárias]]. **A confusão entre eles é
a maior fonte de erro do projeto — e um deles é produção.**

| Container | Onde | Porta no host | Banco | O que é |
|---|---|---|---|---|
| **`db-vida`** | CET `10.39.64.110` | **5432** | `vida` | 🚨 **PRODUÇÃO** |
| `db-vida-cet-dev` | CET `10.39.64.110` | **5433** | `vidadev` | cópia de out/2025 — usamos como teste |
| `vida-rio-dev` | Proxmox `10.29.0.167` | 5432 | `vida` | desenvolvimento |

> 🚨 **O `db-vida` é o banco de produção.** O `.env` da API em `Vida-rio` aponta
> para `DB_HOST=10.39.64.110`. Não é homologação, apesar do rótulo no `.env` de
> dev dizer isso.
>
> **Nada de escrita ali sem consciência do que está fazendo.** Os scripts Python
> vêm configurados com `localhost:5432`, que naquela máquina **é o `db-vida`** —
> foi assim que a carga de out/2025 foi feita, direto em produção.

**Antes de rodar qualquer coisa na máquina do CET, confirme a porta:**

```powershell
docker port db-vida            # 5432  ← produção
docker port db-vida-cet-dev    # 5433  ← teste
```

A armadilha é o container **sem** "dev" no nome estar na porta 5432, que é a
padrão — qualquer script copiado de outro lugar cai em produção por omissão.

> A armadilha: no CET, o container **sem** "dev" no nome é o que está na porta
> **5432**. Script apontado para `localhost:5432` grava na homologação, não no
> dev. Conferir sempre com `docker port <container>`.

## Nomes que se repetem

Três coisas quase homônimas, e não são a mesma:

| Nome | O que é |
|---|---|
| `VIDA-RIO-DEV` (maiúsculo) | hostname da **VM** no Proxmox |
| `vida-rio-dev` (minúsculo) | **container** do PostGIS dentro dela |
| `DB_HOST=vida-rio-dev` | aponta para o container acima — **só resolve dentro da rede Docker da VM** |

O `.env` do repositório usa o nome do container. Copiado para a máquina local,
ele **não funciona**: `getaddrinfo ENOTFOUND vida-rio-dev`.

---

## Topologia de rede

```
┌─ Sua máquina ───────────────────────────────────────────┐
│  SSH 22 → 10.29.0.167     ✅                             │
│  TCP  * → 10.39.64.110    ❌ bloqueado (inclusive 3389)  │
└──────────────────────────────────────────────────────────┘

┌─ VM Proxmox · VIDA-RIO-DEV · 10.29.0.167 ───────────────┐
│  api :8000 · front :3000 · nginx :80/443                 │
│  vida-rio-dev (postgis 17-3.5-alpine) :5432              │
│  mongodb :27017 · redis :6379   ← declarados, sem uso    │
│                                                          │
│  TCP → 10.39.64.110  ❌ bloqueado                         │
└──────────────────────────────────────────────────────────┘

┌─ Workstation Windows do CET · 10.39.64.110 ─────────────┐
│  rio.rj.gov.br · rede 10.39.64.0/22 · gw 10.39.64.41     │
│  Docker Desktop (PERSONAL, não logado)                   │
│  db-vida :5432   ·   db-vida-cet-dev :5433               │
└──────────────────────────────────────────────────────────┘
```

**Não há rota entre os segmentos.** Diagnosticado em 02/09/2026:

```bash
nc -z -w 3 -v 10.39.64.110 5432   # timeout
nc -z -w 3 -v 10.39.64.110 3389   # timeout  ← nem o RDP passa
tracepath -m 8 10.39.64.110       # atravessa 8 hops do backbone 172.16.x
```

O tráfego **é roteado** — atravessa o backbone da Prefeitura — mas o firewall
descarta na chegada. Como nem a 3389 responde, não é filtro de porta: é bloqueio
de todo TCP daquela origem para aquele host.

> O `ping` retorna `Redirect Host (New nexthop: 10.29.0.33)` e 100% de perda.
> Isso é normal: ICMP echo é filtrado por política. Não indica falta de rota.

Para liberar, é chamado com a TI do CET, com esta informação:

> Origem `10.29.0.167` (VLAN 10.29.0.0/x), destino `10.39.64.110:5432`. Rota
> anunciada via ICMP Redirect para next hop `10.29.0.33`. ICMP e TCP sem
> resposta. Solicito liberação para acesso de desenvolvimento à base de
> homologação.

---

## Conectar no banco do Proxmox (DBeaver)

A 5432 da VM é **recusada** de fora (firewall local na VM), mas a **22 está
aberta**. Vai por túnel SSH — que é o jeito certo, sem expor banco nenhum.

**Aba Principal:**

| Campo | Valor |
|---|---|
| Host | `localhost` |
| Porta | `5432` |
| Banco | `vida` |
| Usuário | `admin` |
| Senha | ver [[Rotinas diárias]] |

**Aba SSH** (no DBeaver novo fica no botão `+ SSH, SSL, …` no canto superior):

| Campo | Valor |
|---|---|
| Use SSH Tunnel | marcar |
| Host | `10.29.0.167` |
| Port | `22` |
| User | `root` |

> **O `localhost` da aba Principal é dentro da VM**, não a sua máquina. Quem
> resolve o endereço é o túnel. Colocar `10.29.0.167` ali é o erro mais comum —
> dá `Connection refused`.

Erros e causas:

| Erro | Causa |
|---|---|
| `Connection refused` | esqueceu de marcar Use SSH Tunnel, ou pôs o IP na aba Principal |
| `password authentication failed` | credencial do **banco** — conferir com `docker inspect vida-rio-dev` |
| falha de autenticação SSH | credencial do **SSH**, que é outra coisa |

## Túnel pelo terminal

Para `psql`, DataGrip ou apontar a API local:

```bash
ssh -L 15432:localhost:5432 root@10.29.0.167
```

Enquanto a janela estiver aberta:

```bash
psql "postgresql://admin:SENHA@localhost:15432/vida"
```

A porta local é **15432** porque a 5432 já está ocupada pelo `atjewel_postgres`.

---

## Trabalhar na máquina do CET

Só por RDP. E lá **o copiar e colar está bloqueado** — comandos precisam ser
digitados à mão.

Alternativas testadas:

| Caminho | Situação |
|---|---|
| Rede direta | ❌ firewall corporativo |
| Clipboard do RDP | ❌ bloqueado |
| `\\tsclient\C` (redirecionamento de unidade) | ⚠️ retorna "Acesso negado" — provável PowerShell **elevado**; testar em janela não-administrador |
| Screenshot da saída | ✅ funciona |

> Drives `\\tsclient\` são montados no contexto do usuário. Um PowerShell aberto
> como Administrador **não os enxerga** e dá `UnauthorizedAccessException`.

Para reduzir digitação, use a busca no histórico do PowerShell:

- **`Ctrl+R`** e digite parte do comando — busca em sessões antigas, inclusive
  as do dev anterior
- **`F8`** completa o que começa com o texto digitado
- O histórico completo fica em
  `$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

---

## Como o dado vai de um ambiente para outro

⚠️ **Thread aberta — não totalmente mapeada.**

O fluxo pretendido, conforme alinhamento de 02/09/2026:

```
1. migrar no vidadev (CET :5433)     ← ambiente de teste
2. validar com o Caio Torres
3. rodar o mesmo em db-vida (:5432)  ← produção
4. trazer uma cópia para o Proxmox   ← dev
```

O passo 3 é os mesmos scripts com `port=5432` em vez de `5433` — que é
exatamente a configuração original deles. **Por isso a troca de porta é o ponto
mais perigoso do processo.**

### Não parece haver sincronia automática

A evidência aponta para **cópia manual**, via `pg_dump` → `docker cp` →
`pg_restore`:

- a pasta `Desktop\ENTRADA\db-postgres` do CET tem dumps de set/2025 a jun/2026
- o histórico do PowerShell do dev anterior mostra os `docker cp` nos dois
  sentidos
- as bases estão em **estados diferentes**: o Proxmox tem 42.231 (31.051 ISP +
  11.180 COR) e o `vidadev` tinha 31.069 (só ISP). Recortes de momentos
  distintos — assinatura de cópia manual, não de sincronia

> Consequência prática: **o Proxmox já tem a etapa da COR feita**; o `vidadev`
> não tinha.

### O que falta descobrir

Existe algum job automatizado que ninguém mencionou? Verificar:

```bash
# no Proxmox e na produção
crontab -l
ls -la /etc/cron.d/ /etc/cron.daily/
systemctl list-timers --all | head -20
find /root /opt /srv -name "*.sh" -o -name "*.py" 2>/dev/null | head

# na produção — quais bancos o pgAdmin administra
docker inspect pgadmin --format '{{range .Config.Env}}{{println .}}{{end}}'
docker exec pgadmin ls -la /var/lib/pgadmin/
```

O pgAdmin guarda os servidores cadastrados num SQLite em
`/var/lib/pgadmin/pgadmin4.db`. Ver o que está lá revela quais bancos alguém
administrava — e se há algum ambiente ainda não mapeado.

## Cuidados

**Nunca deixe sessão `psql` interativa com transação aberta.** Em 01/09/2026 um
`BEGIN; TRUNCATE ...` sem `COMMIT` segurou lock `ACCESS EXCLUSIVE` por **15
horas** e travou a importação inteira. Ver [[2026-09-02]].

O prompt avisa:

| Prompt | Estado |
|---|---|
| `vidadev=#` | normal |
| `vidadev=*#` | **transação aberta** — falta `COMMIT` ou `ROLLBACK` |
| `vidadev=!#` | transação **abortada** — só `ROLLBACK` destrava |

Para comando único, prefira a forma não interativa, que faz autocommit e fecha
a sessão sozinha:

```powershell
docker exec <container> psql -U admin -d <banco> -c "COMANDO"
```

> Com `-c`, **não** use `BEGIN` sem `COMMIT` na mesma string: ao sair, o psql
> faz rollback e o comando é desfeito.

**Confira sempre o prompt antes de rodar qualquer coisa:**

| Prompt | Onde você está |
|---|---|
| `PS C:\Users\Lucas Barbosa>` | sua máquina |
| `root@VIDA-RIO-DEV:~#` | Proxmox |
| `PS C:\Users\Transitar>` | máquina do CET |
