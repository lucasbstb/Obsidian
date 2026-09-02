# Importar a planilha do ISP

> Procedimento vivo — atualizar a cada carga, principalmente se a planilha mudar
> de formato. Relacionados: [[Acessar os bancos]],
> [[Backup e restore do banco]], [[Arquitetura e mapa do sistema]],
> [[2026-09-02]].

Reconstruído em 02/09/2026 a partir do código, do Timeline do VS Code e de uma
conversa com o Yerlon. **Não havia documentação.**

---

## O que é esse pipeline

Quatro das seis fontes de dados do VIDA **não são importadas pela API**. Elas
entram por scripts Python soltos, em
`C:\Users\Transitar\Desktop\ENTRADA\script-migracao` na máquina do CET.

| Script | Lê | Escreve |
|---|---|---|
| `ordenar_dados.py` | `basedados.xlsx` aba "Vítimas 2022-2025" | `basedados_ordenados.xlsx` |
| `new_file_migration.py` | base do ISP, aba "Regioes" | `regioes.xlsx` |
| `criacao_registros_cbrj.py` | `CBMERJ-2023-2026.xlsx` | tabela `ocorrencias_cbrj` |
| **`migracao.py`** | `basedados_ordenados.xlsx` | `incident_trafficincident`, `incident_victim`, … |
| `migration_data_incidents.py` | *(não analisado)* | |

**`migracao.py` é o principal.** Os outros três atendem fontes ou etapas
diferentes.

> ⚠️ `criacao_registros_cbrj.py` é do **CBMERJ**, não do ISP. As colunas não
> batem — rodá-lo na planilha do ISP quebra em `KeyError`.

## A planilha

`Sinistros_ISP_2022-2025.xlsx`, aba **`_BDAcidentes`**. Desnormalizada: **uma
linha por vítima**, agrupadas por `controle`.

```
controle | data_fato | fx_hora | logradouro | ref_numerica | ref_textual |
intersecao | bairro | crt | coordenadas | latitude | longitude |
dsc_titulo_criminal | chave_vitima | falecido | data_nascimento | idade |
sexo | cor | escolaridade | ...
```

Duas coisas importantes:

**A geometria vem pronta.** `coordenadas` traz EWKB hex do PostGIS
(`0101000020E6100000...` = Point, SRID 4326), inserível direto na coluna `geom`.
**Nenhuma chamada ao Google é necessária** — o bug do geocoding fixado em
Fortaleza não afeta este fluxo.

**As colunas mudaram em relação à planilha antiga.** A antiga tinha
`titulo_crime`, `evento` e `desc_local_fato`; a nova tem `dsc_titulo_criminal` e
não tem as outras duas. Comparar antes de cada carga nova.

---

## ⚠️ Antes de qualquer coisa

**1. Trocar a conexão dos scripts.** Eles vêm apontando para `localhost:5432` /
`vida` — que é o **`db-vida`**, a homologação. Para o ambiente de dev é `5433` /
`vidadev`.

```powershell
cd C:\Users\Transitar\Desktop\ENTRADA\script-migracao
Select-String -Path *.py -Pattern "port=|database="
```

No `migracao.py` a conexão está repetida **dentro de cada função**
(`inserir_incident`, `pegar_cor_pele`, `pegar_escolaridade`…). Não basta trocar
no topo — use `Ctrl+Shift+H` no VS Code.

**2. Fazer o backup.** Ver [[Backup e restore do banco]].

**3. Corrigir o `NaN`.** No `carregar_dados_planilha()`, adicionar:

```python
df = df.where(pd.notna(df), None)
```

Sem isso, células vazias entram no banco como a **string literal "NaN"** — não
como `NULL`. `WHERE campo IS NULL` deixa de funcionar e a tela mostra "NaN" ao
usuário. O `criacao_registros_cbrj.py` já faz isso; o `migracao.py` não fazia.

---

## Limpar o banco

Truncar **só as tabelas transacionais**. As de domínio precisam sobreviver,
porque o script faz lookup de FK por nome nelas
(`SELECT id FROM incident_victim_skin_color WHERE LOWER(name) LIKE '%...%'`).

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "TRUNCATE incident_trafficincident, incident_victim, incident_vehicle, incident_trafficlane, incident_review RESTART IDENTITY CASCADE"
```

O `CASCADE` pega sozinho `incident_image`, `incident_incidentvictiminjury`,
`incident_trafficincidentagent` e `incident_trafficincidentvehicles`.

**Nunca truncar:** `polygon_boundary` (44.534), `neighborhood` (167),
`spatial_ref_sys`, as `pagc_*`/`*_lookup` do geocoder Tiger, as `incident_*` de
domínio e as `auth_*`.

Conferir:

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select (select count(*) from incident_trafficincident) sin,(select count(*) from polygon_boundary) pol"
```

Esperado: `sin = 0`, `pol = 44534`.

> Use a forma `-c`, **não** `psql` interativo com `BEGIN`. Ver o incidente de
> lock em [[2026-09-02]].

---

## Rodar o `migracao.py`

O script é um **rascunho**, não um programa pronto. O `main()` tem blocos
comentados; roda-se **um bloco por vez**, descomentando e comentando de volta.

> A ordem dos blocos é a ordem em que estão escritos. *(confirmado pelo Yerlon)*

| Ordem | Bloco | Linhas aprox. | Escreve em |
|---|---|---|---|
| 1 | Detalhes do Incidente | 643–647 | `incident_trafficincident` |
| 2 | Vítimas | 650–653 | `incident_victim` |
| 3 | Vias | 656–660 | `incident_trafficlane` |
| 4 | Veículos | 663–667 | `incident_vehicle` |
| — | ~~Bairros~~ | 670–673 | ⛔ **pular** |
| 5 | Atualizando colunas | 677–683 | `UPDATE` severity + description |
| 6 | Atualizando horário | 685–689 | `UPDATE occurred_at` |
| — | ~~Listas complementares~~ | 697–704 | ⛔ **pular** |

**Por que pular os dois:**

O bloco de **bairros** insere em `neighborhood`, que já tem 167 linhas — e ele
lê de `regioes.xlsx`, não desta planilha (por isso usa `Bairro` com maiúscula).
Rodar duplicaria os bairros.

O bloco de **listas complementares** popula `incident_victim_skin_color`,
`incident_victim_scholarity` e `incident_victim_marital_status`. Carga única,
já feita.

### O ciclo por bloco

```
1. descomenta o bloco N              (selecionar linhas + Ctrl+/)
2. ativa o break (linha 692)         →  processa 1 sinistro
3. python migracao.py                →  resposta em segundos
4. confere no banco
5. limpa o registro de teste
6. comenta o break                   →  solta nos 31 mil
7. python migracao.py
8. confere a contagem
9. comenta o bloco N, descomenta o N+1
```

**Nunca deixe dois blocos ativos ao mesmo tempo.**

O `break` e os comentários são controles **independentes**: o `break` decide
*quantos* grupos processa; os comentários decidem *quais* operações rodam.

> É exatamente o que o Yerlon fazia. O Timeline do VS Code mostra ele comentando
> o `break` em 27/10/2025 às 11:24 — testava com o break ativo, validava, e só
> então soltava em tudo.

### Limpeza entre teste e rodada completa

| Bloco | Limpeza |
|---|---|
| 1 | `TRUNCATE incident_trafficincident, incident_victim, incident_vehicle, incident_trafficlane, incident_review RESTART IDENTITY CASCADE` |
| 2 | `TRUNCATE incident_victim CASCADE` |
| 3 | `TRUNCATE incident_trafficlane` |
| 4 | `TRUNCATE incident_vehicle` |
| 5, 6 | nenhuma — são `UPDATE`, não duplicam |

Pular a limpeza deixa o primeiro `controle` duplicado.

---

## Conferência

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select (select count(*) from incident_trafficincident) sin,(select count(*) from incident_victim) vit,(select count(*) from incident_trafficlane) via,(select count(*) from incident_vehicle) vei"
```

Referência da base anterior (31.069 sinistros):

```
vítimas    38.435     ≈ 1,24 por sinistro
veículos   38.280     ≈ 1,23 por sinistro
vias       32.740     ≈ 1,05 por sinistro
```

Proporções muito diferentes indicam problema no agrupamento por `controle`.

O comentário na linha 649 do script anota **31051** como o total esperado.

### O bug do `LIKE` — conferir depois do bloco 2

```python
WHERE LOWER(iwatl."name") LIKE '%{cor}%'
```

A query passa a **coluna do banco** para minúscula, mas interpola o valor **sem
converter**. Os valores da planilha vêm em maiúsculas (`PARDA`, `BRANCA`), e os
do banco estão capitalizados (`Parda`, `Branca`). `LOWER(name) LIKE '%PARDA%'`
nunca casa.

Só funciona se quem chama a função fizer `.lower()` antes. Conferir:

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select count(*) total, count(*) filter (where skin_color_id is null) sem_cor from incident_victim"
```

Se `sem_cor` = `total`, o lookup está falhando — e falha **em silêncio**, porque
a função retorna `[]` no erro em vez de estourar.

---

## Se travar

O sintoma é o script imprimir um `=== Número do incidente ===` e parar. Quase
sempre é lock, não lentidão.

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select pid,state,left(query,40) q from pg_stat_activity where state is not null"
```

Procurar **`idle in transaction`** — é uma sessão `psql` esquecida com `BEGIN`
pendente. Matar:

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select pg_terminate_backend(PID)"
```

> Matar o processo Python **não** resolve: o backend do Postgres continua na
> fila esperando o lock.

## Se estiver só lento

É esperado. O `inserir_incident` abre uma **conexão nova ao banco a cada
sinistro** — 31 mil conexões por bloco. Só o custo de abrir e autenticar dá
dezenas de minutos.

Como distinguir:

| Sintoma | Significado |
|---|---|
| `=== Número do incidente ===` rolando | progredindo |
| parado no mesmo por mais de 1 minuto | lock |

Acompanhar por fora, em outro terminal:

```powershell
docker exec db-vida-cet-dev psql -U admin -d vidadev -c "select count(*) from incident_trafficincident"
```

Se aparecer `FATAL: sorry, too many clients already`, as conexões não estão
sendo liberadas. A correção é adicionar `conn.close()` no `finally` das funções
— hoje só o cursor é fechado.

---

## ⚠️ O `accident_code` colide — e a vítima é ligada por ele

Descoberto em 02/09/2026, durante a primeira carga completa.

O script grava no `accident_code` apenas o **prefixo** do `controle`:

```python
vitima["controle"].split("-", 1)[0]
```

E o `controle` tem **formatos diferentes** na planilha. Em parte dos registros o
primeiro segmento é um carimbo de tempo único (`20251231111430`); em outra parte
é só ano+mês (`202207`). Resultado, com 38.592 sinistros importados:

```
total      38.592
distintos  36.967
           ──────
           1.625 códigos repetidos

accident_code  qtd   pri          ult
202207         666   2022-01-01   2022-08-02
202456          44   2022-11-18   2024-05-06
2024518         39   2023-09-14   2024-05-19
```

Não são duplicatas: são sinistros distintos com código colidido. A importação em
si está certa — o `controle` é a chave do `groupby`, então há um sinistro por
`controle`.

**O problema é o que depende desse campo.**

`preparar_dados_traffic_vitimas` liga a vítima ao sinistro assim:

```python
'code': vitima["controle"].split("-", 1)[0]
vitima["incident_id"] = buscar_traffic_incident_id(vitima["code"])
```

Para o código `202207`, a busca devolve **um** dos 666 sinistros. Todas as
vítimas desses 666 vão para o mesmo registro. O mesmo vale para vias e veículos.

E os blocos 5 e 6 fazem `UPDATE ... WHERE accident_code = %s` — atingindo 666
linhas de uma vez, inclusive no `UPDATE` do horário.

### O conserto seria de duas linhas

O `controle` é único. Guardá-lo inteiro resolve tudo:

```python
'code': vitima["controle"]                    # linha ~481
# e o equivalente em preparar_dados_traffic_incident
```

Depois: truncar, refazer bloco 1 e bloco 2. ~30 minutos.

### Decisão de 02/09/2026

**Não mexer no código do dev anterior.** A carga foi feita como está, para
validar o pipeline de ponta a ponta; as inconsistências serão tratadas no banco
depois.

> ⚠️ **Enquanto isso, essa base não serve para análise.** Vítimas, vias e
> veículos de até 666 sinistros diferentes podem estar pendurados num registro
> só. Não tirar conclusão estatística dela até o `accident_code` ser resolvido.

Antes de rodar os blocos 5 e 6, tirar um dump — assim dá para voltar sem refazer
os blocos 1 a 4:

```powershell
docker exec db-vida-cet-dev pg_dump -U admin -d vidadev -Fc -f /tmp/antes_updates.dump
docker cp db-vida-cet-dev:/tmp/antes_updates.dump C:\Users\Transitar\Desktop\ENTRADA\db-postgres\
```

## ⚠️ Cor de pele e escolaridade não são importadas

Confirmado na carga de 02/09/2026, depois do bloco 2:

```
skin_color_id | count        scholarity_id | count
            7 | 47760                   14 | 47760
```

Uma única linha em cada — **todas as 47.760 vítimas no mesmo valor**, que é o
último da tabela ("Não informado"). Estado civil deve ter o mesmo padrão.

A causa tem **duas camadas**:

```sql
WHERE LOWER(iwatl."name") LIKE '%{cor}%'
```

| Planilha | Tabela de domínio | Casa? |
|---|---|---|
| `PARDA` | `Parda` → `parda` | ❌ maiúsculas — `.lower()` resolveria |
| `SEM INFORMAÇÃO` | `Não informado` | ❌ **vocabulários diferentes** |

> Corrigir só o `.lower()` **não basta**. "SEM INFORMAÇÃO" nunca casa com "Não
> informado" por `LIKE` — precisa de um mapa explícito entre os dois
> vocabulários.

E a falha é silenciosa: a função retorna `[]` no erro em vez de estourar.

### É recuperável sem refazer a carga

O `incident_victim.name` guarda o `chave_vitima` da planilha. Dá para voltar
depois casando por ele:

```
planilha[chave_vitima] → cor / escolaridade / estado_civil
    → UPDATE incident_victim WHERE name = chave_vitima
```

Entra na lista de correções no banco, junto com o `accident_code`.

## Dívidas conhecidas deste pipeline

1. **Sem idempotência.** `INSERT` puro, sem `ON CONFLICT`. Rodar duas vezes
   duplica a base. O módulo de Duplicidades da aplicação existe justamente
   porque isso acontece
2. **SQL injection** nas funções `pegar_*` — f-string direto na query
3. **Match ambíguo** — `LIKE '%valor%'` com `fetchone()` e sem `ORDER BY` pega a
   primeira linha que casar. Duas entradas parecidas e a vítima recebe o valor
   errado, em silêncio
4. **Vazamento de conexão** — `conn` nunca é fechada
5. **`UPDATE ... WHERE name = %s`** nas vítimas — nomes repetidos atualizam mais
   de uma linha
6. **`finally: cursor_destino.close()`** — se o `connect()` falhar, dá
   `NameError` e engole a exceção real
7. **Credenciais em texto puro** no código

> A saída de médio prazo é **portar isso para dentro da API** como um terceiro
> importador, junto de SAMU e PRF, que já têm o padrão pronto em
> `importer-samu.service.ts`. Vira endpoint, com validação, transação e log — em
> vez de `.py` na Desktop de uma workstation.
