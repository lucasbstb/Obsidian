# Aplicar migração de banco

> Procedimento vivo — atualizar quando o processo mudar, sem criar versão nova.
> Relacionados: [[2026-08-11 — Integração ERP Safira — Levantamento de Requisitos]],
> [[Deploy do front em produção]], [[2026-08-12]].

Estabelecido em 12/08/2026, depois de instalar o controle de migrações.
Vale para o `at-jewel-back`.

> **Só existem dois ambientes: local e produção.** Não há homolog. Toda mudança
> vai da máquina do Lucas direto para o `AT-AGENTE`.

---

## Regras que não se quebram

**1. Nunca escrever DDL direto em produção.** Toda mudança vira arquivo numerado
em `src/shared/database/migrations/`, e o *mesmo arquivo* percorre os dois
ambientes sem ser editado no caminho.

**2. Migração que já rodou fora do local nunca mais se edita.** Descobriu erro na
26 depois de aplicada? Cria a **27** que corrige. Editar faz os ambientes
divergirem — e o `db:status` acusa, porque o checksum deixa de bater.

**3. Coluna nova em tabela com dado tem que ser nullable** (ou ter default).
`ADD COLUMN ... NOT NULL` sem default falha na hora: o Postgres não sabe o que
pôr nas linhas existentes.

**4. A migração e a entidade TypeORM vão no mesmo commit.** São duas descrições
da mesma tabela mantidas à mão, e nada verifica se concordam. Separar é como o
schema e o código se perdem um do outro.

**5. O marcador no `MANIFESTO` entra junto com a migração.** Regra criada em
14/08/2026, depois de descobrir o `MANIFESTO` parado na 25 na véspera de um
deploy de seis migrações — o `db:verify` mostrava as seis como `??`. Migração
sem marcador é migração que o `verify` não sabe conferir, e o `verify` é o
**único** jeito de provar que produção não divergiu depois de aplicar. Falha
silenciosa: `status` e `migrate` funcionam normalmente, e só o `verify` acusa.

Migração que não cria objeto de schema também tem marcador:

| Se a migração… | tipo | alvo |
|---|---|---|
| cria tabela / coluna / índice / matview / enum | `tabela` `coluna` `indice` `matview` `tipo` | o nome do objeto |
| só torna uma coluna opcional | `nullable` | `tabela.coluna` |
| só amarra tabelas existentes (FK, check, unique) | `constraint` | o nome da constraint |
| só semeia permissão, sem DDL | `permissao` | `PAPEL\|permissao` |

No `permissao` o separador é `|` e não `:`, porque a permissão já contém
dois-pontos.

---

## A ordem no deploy

Depende do tipo da mudança.

### Aditiva — tabela nova, coluna nova nullable, índice novo

```
1. aplica a migração      ← código antigo continua rodando, nem percebe
2. faz deploy do código
```

Sem janela de erro: o código antigo não sabe que a coluna existe. Inverter a
ordem quebra — o código novo procura coluna que ainda não está lá.

### Destrutiva — trocar tipo, renomear, remover

Não tem ordem segura. Fatiar em passos aditivos:

```
1. cria a coluna nova, ao lado da velha       (migração)
2. código passa a escrever nas DUAS           (deploy)
3. copia o histórico da velha para a nova     (migração)
4. código passa a ler só da nova              (deploy)
5. remove a velha                             (migração, semanas depois)
```

É o que `pagamentos_venda.forma_pagamento` vai exigir para sair de ENUM para FK.
Não é uma migração — são três, com deploys no meio.

---

## Passo a passo

### Local

```bash
cd at-jewel-back

# 1. escrever src/shared/database/migrations/NN_nome.sql
# 2. registrar o marcador no MANIFESTO de scripts/migrate.js  (regra 5)

npm run db:status      # confirma que ela aparece como pendente
npm run db:migrate     # aplica
npm run db:verify      # tem que fechar N/N, sem nenhum "??"

# 3. atualizar a *.orm-entity.ts correspondente
# 4. testar
# 5. commit do .sql + .ts + migrate.js juntos, e push
```

### Produção

```bash
ssh "AT-Front/Back"                 # exige VPN — ver [[deploy-producao-atjewel]]
cd ~/at-jewel/at-jewel-back

git status                          # tem que estar limpo — pull sobrescreve
git pull origin main

# backup — obrigatório
docker exec atjewel_postgres pg_dump -U atjewel -d atjewel_dev > ~/backup_antes_NN.sql
ls -lh ~/backup_antes_NN.sql

docker exec atjewel_api npm run db:status     # confere o que está pendente
docker exec -it atjewel_api npm run db:migrate # -it: pede confirmação digitada

docker restart atjewel_api          # só depois da migração
```

Os tres containers em producao, porque o nome nao e obvio e o do front ja fez um
`docker restart` falhar com `No such container`:

```
atjewel_app        front   porta 80
atjewel_api        back    porta 3000
atjewel_postgres   banco   porta 5432
```

**A ordem `git pull` -> `db:migrate` e corrida, sem intervalo.** O back roda
`start:dev` com bind mount: assim que o pull termina, o watch recompila e o
codigo novo entra no ar sozinho, procurando colunas que a migracao ainda nao
criou. Sao segundos de instabilidade, mas e melhor saber.

O `-it` é obrigatório: sem TTY o script aborta em vez de aplicar sem confirmação.

---

## Os comandos

| Comando | O que faz | Escreve? |
|---|---|---|
| `npm run db:status` | aplicadas, pendentes, e avisa se um arquivo aplicado foi editado | não |
| `npm run db:verify` | checa no schema se cada migração de fato rodou | não |
| `npm run db:baseline` | carimba as existentes como aplicadas, sem executar | sim |
| `npm run db:migrate` | aplica as pendentes, em ordem, uma transação cada | sim |

`baseline` é operação de uma vez por ambiente — já feito em local e produção em
12/08/2026. Só volta a ser necessário em ambiente novo que já tenha schema.

---

## Cuidados de ambiente

**Local e produção são quase idênticos:** mesmo banco (`atjewel_dev`), mesmo
container (`atjewel_postgres`), mesmo usuário, mesma porta. **Só o host difere.**
Um comando copiado de um terminal para o outro roda igual nos dois.

Proteções em vigor:

- O `.env` de produção tem `AMBIENTE=producao`; o script mostra `[PRODUCAO]` no
  cabeçalho e exige que se digite `PRODUCAO` antes de escrever
- `NODE_ENV` **não** distingue nada — o compose fixa `development` nos dois
- O prompt do servidor (`root@AT-AGENTE:`) é o sinal visual; conferir antes de
  rodar qualquer `psql`

**As dependências vivem no container.** `node_modules` é volume nomeado, então
rodar `npm run db:*` no host do servidor falha com `Cannot find module 'pg'`.
Sempre `docker exec atjewel_api`.

**O `.env` do servidor não pode ser perdido.** Contém `DATABASE_URL`,
`JWT_SECRET`, `ENCRYPTION_KEY`, `HASH_SECRET` e `SAFIRA_API_KEY`. Perder o
`HASH_SECRET` em particular inviabiliza encontrar cliente por telefone — os
hashes gravados deixam de bater. Editar pelo VS Code, nunca por redirecionamento
de shell (`>` no lugar de `>>` apaga o arquivo).

---

## Se der errado

**Migração falhou no meio.** Não fez nada: cada uma roda em transação e o
Postgres tem DDL transacional. O registro só entra no mesmo `COMMIT`. Corrige o
arquivo e roda de novo — retoma de onde parou.

**Migração aditiva aplicada por engano.** `DROP COLUMN` / `DROP TABLE` resolve,
já que ninguém preencheu nada ainda. Depois, apagar a linha correspondente em
`schema_migrations`.

**Migração destrutiva deu errado.** Restaurar o `pg_dump`, com o sistema parado.
É o motivo de separar aditivo de destrutivo antes de escrever.
