# Backup e restore do banco

> Procedimento vivo — atualizar quando o processo mudar.
> Relacionados: [[Acessar os bancos]], [[Importar a planilha do ISP]],
> [[2026-09-02]].

Estabelecido em 02/09/2026, antes da primeira carga do ISP.

---

## Por que importa mais do que parece

Os containers do CET usam **volumes anônimos** — nomes como
`7a38f39305e05906258c47e0b3a94f6f1d00d197a88c84a891979b14452b82bb`. Eles
sobrevivem a `docker rm`, mas:

| Comando | Efeito no dado |
|---|---|
| `docker rm db-vida` | volume sobrevive (fica órfão) |
| `docker rm -v db-vida` | volume **apagado** |
| `docker volume prune` | apaga todos os órfãos — **inclui esses** |
| Docker Desktop → "Clean / Purge data" | apaga tudo |

Ninguém olhando aquela lista de hashes sabe que ali está a base do VIDA. Um
`volume prune` de rotina leva junto.

E a máquina é uma **workstation Windows**, com Docker Desktop PERSONAL, sem
`--restart` nos containers. Ela é ligada e desligada todo dia.

---

## Backup

Roda na máquina do CET, via RDP.

```powershell
# 1. gerar o dump dentro do container
docker exec db-vida-cet-dev pg_dump -U admin -d vidadev -Fc -f /tmp/vidadev_DDMMAAAA.dump

# 2. trazer para o Windows
docker cp db-vida-cet-dev:/tmp/vidadev_DDMMAAAA.dump C:\Users\Transitar\Desktop\ENTRADA\db-postgres\

# 3. conferir
dir C:\Users\Transitar\Desktop\ENTRADA\db-postgres
docker exec db-vida-cet-dev pg_restore -l /tmp/vidadev_DDMMAAAA.dump | Select-Object -First 15
```

O `pg_restore -l` lista o índice do dump. Se listar tabelas, o arquivo está
íntegro — fazer isso **antes** de confiar no backup.

Para o `db-vida`, trocar container e banco: `db-vida` / `vida`.

> **`pg_dump` não pede parada.** Usa snapshot transacional; o backup sai
> consistente com o banco em uso.

### ⚠️ Nunca redirecionar no PowerShell

```powershell
docker exec db-vida pg_dump ... -Fc > C:\temp\vida.dump    # ❌ CORROMPE
```

O PowerShell trata a saída como texto, converte encoding e troca LF por CRLF. O
dump fica inutilizável e **você só descobre na hora de restaurar**. Por isso o
`-f` dentro do container seguido de `docker cp` — esse caminho é binário-seguro.

### Se pedir senha

```powershell
docker exec -e PGPASSWORD=SENHA db-vida-cet-dev pg_dump -U admin -d vidadev -Fc -f /tmp/x.dump
```

### Levar também os globals

Usuários e senhas do Postgres **não** vão no `pg_dump` de um banco só:

```powershell
docker exec db-vida-cet-dev pg_dumpall -U admin --globals-only -f /tmp/globals.sql
docker cp db-vida-cet-dev:/tmp/globals.sql C:\Users\Transitar\Desktop\ENTRADA\db-postgres\
```

---

## Restore

```powershell
docker cp C:\...\vidadev_DDMMAAAA.dump db-vida-cet-dev:/tmp/
docker exec db-vida-cet-dev pg_restore -U admin -d vidadev --clean --if-exists --no-owner /tmp/vidadev_DDMMAAAA.dump
```

Para restaurar **no Proxmox**, num banco novo (nunca por cima do `vida`):

```bash
docker cp vidadev.dump vida-rio-dev:/tmp/
docker exec -it vida-rio-dev psql -U admin -c "CREATE DATABASE vida_cet"
docker exec -it vida-rio-dev psql -U admin -d vida_cet -c "CREATE EXTENSION IF NOT EXISTS postgis"
docker exec -it vida-rio-dev pg_restore -U admin -d vida_cet --no-owner --no-privileges /tmp/vidadev.dump
```

Duas coisas obrigatórias:

**Criar a extensão PostGIS antes.** Sem ela, toda coluna `geometry` falha no
restore.

**`--no-owner --no-privileges`.** Evita erro de roles que não existem no destino.

> Os dois ambientes usam a **mesma imagem base** (`postgis/postgis:17-3.5-alpine`),
> então não há problema de compatibilidade de versão entre eles.

---

## Como identificar um dump antigo

A pasta `db-postgres` do CET tem dumps de vários períodos, sem indicação de
origem. O cabeçalho do arquivo guarda o nome do banco:

```powershell
docker cp C:\...\backup_15_06_2026.dump db-vida-cet-dev:/tmp/
docker exec db-vida-cet-dev pg_restore -l /tmp/backup_15_06_2026.dump | Select-Object -First 12
```

Procurar a linha `dbname:`:

- `dbname: vida` → veio do **`db-vida`** (homologação)
- `dbname: vidadev` → veio do **`db-vida-cet-dev`**

O cabeçalho também traz a versão do Postgres de origem e a data exata do dump.

### O histórico que já existe

```
18/09/2025    3.3 MB  ┐
19/09/2025    3.4 MB  ├─ base pequena
20/10/2025    3.4 MB  ┘
21/10/2025   63.0 MB  ← a carga grande aconteceu aqui
27/10/2025   64.4 MB
15/04/2026   64.5 MB  (dois arquivos no mesmo dia — provavelmente um de cada banco)
15/06/2026   65.4 MB
01/09/2026   64.4 MB  ← vidadev_01092026.dump, gerado por nós
```

O salto entre 20 e 21/10/2025 é a importação em lote das planilhas.

E o `vidadev_01092026.dump` ter praticamente o mesmo tamanho do
`backup_27_10_2025.dump` (4 KB de diferença) mostra que o **`vidadev` estava
congelado desde outubro de 2025** — ninguém escreveu nada nele desde então.

---

## Cópia física do volume

Alternativa ao dump lógico, para segurança extra:

```powershell
docker run --rm -v <hash_do_volume>:/data -v C:\temp:/backup alpine tar czf /backup/volume.tar.gz -C /data .
```

Descobrir o hash:

```powershell
docker inspect db-vida-cet-dev --format "{{json .Mounts}}"
```

> Só restaura em Postgres da **mesma versão maior**. Para mover dado entre
> ambientes, use sempre o `pg_dump`.

---

## Se der errado

**Dump corrompido.** Quase sempre é redirecionamento do PowerShell. Refazer com
`-f` + `docker cp`.

**Restore reclama de extensão.** Criar `postgis` no banco de destino antes.

**Restore reclama de role inexistente.** Faltou `--no-owner --no-privileges`.

**O arquivo ainda está na máquina do CET.** É o problema em aberto: sem rota de
rede e sem clipboard, a transferência depende do redirecionamento de unidade do
RDP. Ver [[Acessar os bancos]].
