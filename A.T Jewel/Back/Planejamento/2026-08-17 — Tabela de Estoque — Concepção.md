# 2026-08-17 — Tabela de Estoque — Concepção

> **Conversa em andamento, interrompida antes de qualquer código.** Nada foi
> criado: nem migração, nem entidade, nem rota. Este documento é o retrato do
> ponto em que paramos.
>
> Relacionados: [[2026-08-11 — Integração ERP Safira — Levantamento de Requisitos]],
> [[2026-08-14]], [[Aplicar migração de banco]].

---

## O modelo do ERP, como o Lucas descreveu

Chave de quatro partes apontando para uma quantidade:

```
empresa - grupo estoque - estoque(local) - codproduto - quant.

001 - Consignado(001)      - Estoque 01 (Armário 01) - 2010 - 02
001 - Disponivel           - Estoque 02 (Armário 02) - 2010 - 08
001 - Consignado_Cliente   - Ana                     - 2010 - 01
```

**Os valores são ilustrativos** — o Lucas confirmou que inventou os nomes para
explicar a estrutura. O que vale é o formato, não os rótulos.

É o padrão de saldo por localização, e se sustenta no nosso CRM.

---

## Tradução das quatro dimensões

O `grupo` **não agrupa lugares** — descreve a *situação* do saldo. Só o `local` é
lugar de fato. Vale batizar assim no nosso modelo, senão daqui a seis meses
ninguém lembra por que as duas colunas são coisas diferentes.

| Dimensão | O que responde |
|---|---|
| `empresa` | de quem é |
| `grupo` | em que situação está |
| `local` | onde fisicamente, ou com quem |
| `produto` | o quê |

---

## Desenho proposto

Duas tabelas de apoio, no padrão dos cadastros que subiram em 14/08 —
`codigo_erp` único, `nome`, `ativo`:

```
grupos_estoque    Disponível · Consignado · Consignado_Cliente
locais_estoque    Armário 01 · Armário 02 · ...
```

E a de saldo:

```sql
estoque (
  id                UUID PK
  empresa_id        FK empresas
  grupo_estoque_id  FK grupos_estoque
  local_estoque_id  FK locais_estoque
  produto_id        FK produtos
  quantidade        INT NOT NULL
  atualizado_em     TIMESTAMPTZ
  UNIQUE (empresa_id, grupo_estoque_id, local_estoque_id, produto_id)
)
```

**A `UNIQUE` das quatro colunas é o coração do desenho.** Garante uma linha por
combinação e permite o `INSERT ... ON CONFLICT DO UPDATE` da sincronização: o ERP
manda a foto, sobrescrevemos a quantidade sem duplicar.

**O `atualizado_em`** responde *"esse número é de quando?"*. Sem ele, saldo velho
e saldo novo têm a mesma cara, e uma sincronização que parou de rodar não é
percebida.

---

## A colisão com `consignacoes`

A terceira linha do exemplo — `Consignado_Cliente · Ana` — **já tem dono no nosso
banco**. A tabela `consignacoes`, da migração 24, guarda o que o ERP não guarda:

```
produto_id · quantidade · destino_tipo (CLIENTE|VENDEDORA)
cliente_id · vendedora_id · status (ABERTA|DEVOLVIDA|VENDIDA) · data_saida
```

**No ERP a consignação é um lugar. No nosso CRM é um evento com ciclo de vida.**

Dois problemas concretos de importar aquela linha como saldo:

1. **"Ana" não é um lugar.** A coluna `estoque` do ERP mistura móvel (`Armário
   01`) com pessoa (`Ana`). Lá convive porque tudo é texto; aqui cliente e
   vendedora são tabelas com UUID, e gravar `"Ana"` como texto perderia o
   vínculo — não daria para responder *"o que a Ana está com nossa peça?"* sem
   casar por nome.
2. **A linha não diz se Ana é cliente ou vendedora.** A nossa `consignacoes` diz.

### Proposta: dois donos, sem conciliação automática

- **`estoque`** responde *"quantas peças, onde, agora"* — **é do ERP**
- **`consignacoes`** responde *"desde quando, com quem, em que situação"* — **é nosso**

Elas se encontram, mas nenhuma sobrescreve a outra. O que não pode existir é uma
terceira coisa tentando conciliá-las sozinha.

---

## Perguntas em aberto — nada avança sem elas

**1. Saldo ou movimento?** O exemplo parece **saldo** — a foto de agora.

- *Saldo:* guardamos um retrato e sobrescrevemos a cada sincronização. Simples,
  mas sem histórico e sem como auditar divergência.
- *Movimento:* guardamos os lançamentos e o saldo é consequência. Mais trabalho,
  porém auditável.

Aberta com o Alessandro desde 11/08.

**2. Quem é o dono do estoque?** Não é pergunta técnica, é de operação. Se o ERP
for o dono — provável, por ser o sistema fiscal —, então a tabela é **espelho**, o
CRM **nunca escreve nela**, nenhuma tela nossa altera quantidade, e em caso de
divergência o ERP está certo por definição. Isso elimina a classe inteira de bug
em que dois sistemas se corrigem em círculo.

**3. A integração é de mão única ou dupla?** Caso real: a vendedora devolve a peça
e marca `DEVOLVIDA` no nosso painel. O ERP só saberá quando alguém lançar lá — até
a próxima sincronização o saldo continua dizendo que está com a Ana. Isso é
aceitável, ou a devolução no CRM precisa virar lançamento no ERP?

As três vão juntas para o Alessandro.

---

## Contexto que já estava registrado

- **`produtos.estoque_atual`** existe desde a migração 16, é um inteiro único por
  produto, **sem dimensão de local** — e não tem escritor pelo webhook do ERP
  (`ErpProdutoDto` não traz o campo), então está em `0` em produção. Precisa ser
  decidido o que fazer com ele quando `estoque` existir: aposentar, ou passar a
  ser o total derivado.
- **`empresas`** foi criada na migração 27 e está vazia.
- **`grupos_estoque`** chegou a ser escrita e testada em 13/08, e foi apagada a
  pedido do Lucas, para voltar quando a modelagem estivesse madura. Volta aqui.
