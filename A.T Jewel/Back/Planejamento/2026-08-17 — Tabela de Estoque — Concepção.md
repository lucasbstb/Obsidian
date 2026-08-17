# 2026-08-17 — Tabela de Estoque — Concepção

> **Nada foi criado ainda.** Nem migração, nem entidade, nem rota. Este é o
> retrato da concepção, atualizado depois da segunda conversa do dia (tarde),
> em que apareceu a partida dobrada e o desenho ganhou forma final.
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

## Partida dobrada — a descoberta que mudou o modelo

O ERP lança estoque em **duas pernas**. Ao pegar uma peça consignada do
fornecedor:

```
+1   no meu estoque          (a peça está comigo)
-1   no fornecedor 1          (eu devo essa peça a ele)
```

**O negativo não é erro: é a obrigação.** Isso derruba de vez a ideia de um
`CHECK (quantidade >= 0)` — a primeira consignação de fornecedor faria a
sincronização inteira falhar. A tabela é espelho, e espelho não corrige.

E revela o que a coluna `local` realmente é.

---

## A dimensão `local` não é lugar — é contraparte

A mesma coluna do ERP guarda coisas de naturezas diferentes:

| Valor | O que é de fato |
|---|---|
| `Armário 01` | lugar físico nosso |
| `Ana` | uma pessoa — cliente ou vendedora |
| `Fornecedor 1` | uma empresa a quem devemos |

No ERP convive porque tudo é texto. No nosso banco, fornecedores (migração 26),
clientes e vendedoras são tabelas com UUID. Gravar `"Fornecedor 1"` como texto
perderia o vínculo — e não daria para responder *"o que eu devo a esse
fornecedor?"* nem *"o que a Ana está com nossa peça?"* sem casar por nome.

**Decisão do Lucas (17/08, tarde): FK real.** Quando é consignação de cliente,
informa-se o id do cliente; de fornecedor, o id do fornecedor.

---

## Decisões tomadas

**Grupos e locais ficam em tabelas SEPARADAS.** Foi avaliada a alternativa de
uma tabela só com discriminador (`tipo 1 = grupo`, `tipo 2 = local`), já que a
estrutura é idêntica. Descartada porque a FK deixaria de impedir a troca: o
banco aceitaria um *local* na coluna de grupo, em silêncio. Daria para recuperar
a garantia com `UNIQUE (id, tipo)` e FK composta, mas o custo de manter duas
tabelas é baixo — são seis colunas cada, e o projeto já tem cinco cadastros
idênticos. Além disso **grupo e local não são a mesma coisa**: grupo descreve a
*situação* do saldo, local descreve *com quem está*. Estrutura igual hoje é
coincidência, não parentesco.

**Sem `CHECK (quantidade >= 0)`** — ver partida dobrada.

**Contraparte com FK real**, uma coluna por tipo, com CHECK de exatamente uma
preenchida.

---

## Desenho proposto

Duas tabelas de apoio, no padrão dos cadastros que subiram em 14/08 —
`codigo_erp` único, `nome`, `ativo`, timestamps:

```
grupos_estoque    Disponível · Consignado · Consignado_Cliente
locais_estoque    Armário 01 · Armário 02 · ...
```

E a de saldo (Postgres 16 confirmado em produção):

```sql
CREATE TABLE estoque (
  id               UUID    PRIMARY KEY DEFAULT gen_random_uuid(),
  empresa_id       UUID    NOT NULL REFERENCES empresas(id),
  grupo_estoque_id UUID    NOT NULL REFERENCES grupos_estoque(id),
  produto_id       UUID    NOT NULL REFERENCES produtos(id),

  -- Contraparte: exatamente UMA. FK de verdade em cada uma.
  local_estoque_id UUID REFERENCES locais_estoque(id),
  fornecedor_id    UUID REFERENCES fornecedores(id),
  cliente_id       UUID REFERENCES clientes(id),
  vendedora_id     UUID REFERENCES vendedoras(id),

  -- Negativo e ESTADO VALIDO: e o que a casa deve.
  quantidade       INTEGER     NOT NULL DEFAULT 0,
  atualizado_em    TIMESTAMPTZ NOT NULL DEFAULT now(),

  CONSTRAINT chk_estoque_contraparte CHECK (
      (local_estoque_id IS NOT NULL)::int
    + (fornecedor_id    IS NOT NULL)::int
    + (cliente_id       IS NOT NULL)::int
    + (vendedora_id     IS NOT NULL)::int = 1
  ),

  -- Derivadas pelo banco: nunca saem de sincronia com as colunas acima.
  contraparte_tipo TEXT GENERATED ALWAYS AS (
    CASE
      WHEN local_estoque_id IS NOT NULL THEN 'LOCAL'
      WHEN fornecedor_id    IS NOT NULL THEN 'FORNECEDOR'
      WHEN cliente_id       IS NOT NULL THEN 'CLIENTE'
      ELSE 'VENDEDORA'
    END) STORED,
  contraparte_id UUID GENERATED ALWAYS AS (
    COALESCE(local_estoque_id, fornecedor_id, cliente_id, vendedora_id)) STORED,

  CONSTRAINT uq_estoque_chave
    UNIQUE (empresa_id, grupo_estoque_id, produto_id, contraparte_tipo, contraparte_id)
);
```

### Por que as colunas geradas

A `UNIQUE` precisa das quatro dimensões, mas três das colunas de contraparte
estão sempre nulas — e **no Postgres nulos nunca colidem entre si**, então a
restrição não pegaria nada. As colunas geradas colapsam a contraparte em
`(tipo, id)`, a `UNIQUE` volta a funcionar e o
`ON CONFLICT ON CONSTRAINT uq_estoque_chave` da sincronização se sustenta:

```sql
INSERT INTO estoque (...) VALUES (...)
ON CONFLICT ON CONSTRAINT uq_estoque_chave
DO UPDATE SET quantidade = EXCLUDED.quantidade, atualizado_em = now();
```

O ERP manda a foto quantas vezes quiser; nunca duplica. Ninguém escreve nas
geradas — o banco calcula.

### O `atualizado_em`

Responde *"esse número é de quando?"*. Sem ele, saldo de hoje e saldo de três
semanas atrás têm a mesma cara, e uma sincronização que parou de rodar fica
invisível. Não é hipótese: em 17/08 a sessão do WhatsApp ficou duas semanas fora
do ar exatamente por não haver nada observando — ver
[[Reconectar o WhatsApp da Anastasia (WAHA)]].

### Efeito colateral a comunicar

Sem `ON DELETE`, o Postgres **impede apagar cliente ou vendedora que tenha
saldo**. Como no A.T. Jewel clientes e vendedoras são apagados de verdade (não
desativados), vira uma regra nova de operação: *não dá para apagar a Ana
enquanto ela estiver com uma peça nossa*. Está correto — apagar deixaria saldo
órfão —, mas alguém vai esbarrar nisso.

---

## A colisão com `consignacoes`

A `consignacoes` (migração 24) guarda `destino_tipo` com `CLIENTE` e
`VENDEDORA`: ela só sabe tratar consignação **de saída**, peça que sai daqui.

A consignação de fornecedor é o oposto — peça que **chega** e que devemos. Hoje
isso não tem lugar nenhum no modelo. Fica em `estoque`, como saldo negativo na
contraparte fornecedor; a `consignacoes` mantém o papel dela, que é o evento com
ciclo de vida (status, data de saída, vínculo).

**No ERP a consignação é um lugar. No nosso CRM é um evento.** Dois donos, sem
conciliação automática:

- **`estoque`** responde *"quantas peças, onde, agora"* — **é do ERP**
- **`consignacoes`** responde *"desde quando, com quem, em que situação"* — **é nosso**

O que não pode existir é uma terceira coisa tentando conciliá-las sozinha.

---

## Perguntas em aberto — vão para o Alessandro

**1. Saldo ou movimento?** O exemplo parece **saldo** — a foto de agora. Se a
resposta for movimento, esta tabela vira consequência de outra
(`estoque_movimentos`) e o saldo passa a ser uma view. Como espelho do que o ERP
mostra hoje, o desenho acima se sustenta sozinho. Aberta desde 11/08.

**2. Quem é o dono do estoque?** Não é pergunta técnica, é de operação. Se o ERP
for o dono — provável, por ser o sistema fiscal —, a tabela é **espelho**, o CRM
**nunca escreve nela**, nenhuma tela nossa altera quantidade, e em divergência o
ERP está certo por definição. Isso elimina a classe inteira de bug em que dois
sistemas se corrigem em círculo.

**3. A integração é de mão única ou dupla?** Caso real: a vendedora devolve a
peça e marca `DEVOLVIDA` no nosso painel. O ERP só saberá quando alguém lançar
lá. Aceitável, ou a devolução no CRM precisa virar lançamento no ERP?

**4. Toda entrada tem contrapartida, ou só a consignação?** *(nova, 17/08)* Se
toda movimentação for lançada nas duas pernas, a soma de todas as linhas de um
produto tende a zero — e isso vira uma **verificação de integridade de graça**:
soma diferente de zero significa sincronização incompleta ou lançamento perdido.
Se só a consignação tiver contrapartida, o modelo é parcial e a conferência não
vale.

---

## Contexto que já estava registrado

- **`produtos.estoque_atual`** existe desde a migração 16, é um inteiro único por
  produto, **sem dimensão de local**, e não tem escritor pelo webhook do ERP
  (`ErpProdutoDto` não traz o campo) — está em `0` em produção. Quando `estoque`
  existir, ele vira ambiguidade: dois lugares dizendo quantidade. Aposentar, ou
  passar a ser total derivado. A recomendação é aposentar.
- **`empresas`** foi criada na migração 27 e está vazia.
- **`grupos_estoque`** chegou a ser escrita e testada em 13/08, e foi apagada a
  pedido do Lucas, para voltar quando a modelagem estivesse madura. Volta aqui.
- **A Elena depende disto.** Nos Planos de Ação ela é *Elena Stockroom —
  catálogo/estoque*. O piloto de 17/08 deu voz a ela no WhatsApp; o assunto dela
  ainda não existe no banco.

---

## Próximo passo

Escrever `32_estoque.sql` com as três tabelas e a entrada no MANIFESTO do
`migrate.js` (marcador `tabela`, alvo `estoque`). A decisão saldo × movimento
não bloqueia: como espelho, o desenho se sustenta.
