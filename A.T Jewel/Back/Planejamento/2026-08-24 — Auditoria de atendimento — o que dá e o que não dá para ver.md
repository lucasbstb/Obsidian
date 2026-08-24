# Auditoria de atendimento — o que dá e o que não dá para ver

**24/08/2026.** Nasceu do pedido do Thiago na ata de 21/08: *"tela de auditoria
que permita ao gestor acompanhar o histórico de interações entre a agente,
Anastasia, e as vendedoras… visualizar diálogos em tempo real."*

Ao levantar o que existe, a conclusão mudou o escopo. Este documento guarda o
porquê, porque a pergunta vai voltar.

---

## A pergunta que parece uma só, mas são três

| Conversa | Quem fala | Capturável? |
|---|---|---|
| Painel | staff ↔ Elena / Anastasia | **sim** — não gravamos ainda |
| WhatsApp interno | vendedora ou ADM ↔ Elena / Anastasia | **sim** — não gravamos ainda |
| **Cliente ↔ vendedora** | duas pessoas | **não, e não há caminho** |

As duas primeiras são a mesma engrenagem e saem juntas — painel e WhatsApp já
compartilham `FerramentasGestaoService` e `FerramentasVendedoraService`.

A terceira é a que o Thiago quer ver, e é a que não existe.

---

## Por que o diálogo cliente↔vendedora não é capturável

**A vendedora é externa e usa o número pessoal dela** (decisão do Lucas,
24/08). O número conectado ao WAHA existe **só para ela falar com a IA**. O
contato com o cliente acontece entre dois telefones que não passam por lugar
nenhum nosso.

O handoff é onde a trilha morre: o aviso que ela recebe **não traz o telefone
do cliente** — ela busca o contato e fala do celular dela.

Três caminhos foram considerados, e dois descartados:

**Conectar o WhatsApp de cada vendedora ao WAHA.** Tecnicamente funciona; o
WAHA aceita várias sessões. Passaríamos a ver toda conversa de cliente dela —
e **toda conversa pessoal dela**, e de terceiros que nunca souberam de nada.
É o problema já documentado em [[waha-le-a-conta-inteira]] multiplicado por
oito pessoas. Não é problema técnico, é de consentimento.

**Caixa de entrada compartilhada (Chatwoot).** É a resposta padrão do mercado,
e o Chatwoot já está provisionado. Mas contraria o modelo: a vendedora é
externa, atende do celular dela, não de uma tela da loja.

**Capturar o desfecho, não o diálogo.** É o que já existe e já funciona.

---

## O que sobrou — e é mais do que parecia

O **relato** é a janela para a conversa que nunca vamos ver, e ele já estava
construído desde 20/08:

- guarda **a frase dela**, cifrada — não um resumo do modelo, porque resumo
  alucina
- ao lado, os campos extraídos: contatou, resultado, remarcado para quando
- se não conseguiu falar, o sistema volta a perguntar em 48h, no máximo duas
  vezes, e depois encerra por inatividade

A unidade da auditoria, então, **não é "o diálogo" — é o atendimento**:

```
encaminhado → vendedora avisada → contato agendado
  → lembrete 15 min antes → cobrança 60 min depois
    → RELATO → desfecho (venda / sem venda / inatividade)
```

### E o sinal mais útil não é nem um nem outro

É o **silêncio**: quem foi avisado e não respondeu · quem prometeu e não
relatou · quem está em segunda retomada · atendimento aberto e parado.

Esse é o SLA da pessoa. É o que um gestor abre às 8h e age em cima — e sai do
`status` das interações, sem coluna nova.

---

## O que dizer ao Thiago

**Não prometer o diálogo cliente↔vendedora.** A auditoria entrega:

> *o atendimento está acontecendo, com quem, em que prazo, e quem está devendo
> resposta* — mais o diálogo completo com as agentes.

Isso responde a pergunta de gestão que ele fez, e é defensável de dizer ao
cliente numa auditoria. O outro caminho não seria.

---

## Construído em 24/08

- migração 38: view `vw_atendimentos_auditoria` + permissão `atendimentos:read`
- `GET /atendimentos`, `/resumo` e `/:id`
- tela `/admin/auditoria`
- ferramenta `feedbacks_de_vendedora` para a Anastasia

## Ainda não construído

1. **Vínculo atendimento↔venda** — a peça mais valiosa que falta. Sem ela a
   tela conta o esforço e não conta o resultado: ticket médio e conversão
   dependem dele
2. **Gravar a conversa com as agentes** — a tabela `conversas` existe desde a
   migração 17, herdada do backend paralelo, e está **vazia**. Precisa de
   coluna para o usuário do painel (só há `cliente_id` e `vendedora_id`), de um
   método `atualizar` no repositório, e de uma decisão sobre gravar ou não as
   chamadas de ferramenta
3. **A fila do que está devendo** — a tela é retrospectiva hoje
4. **Canal do contato** (WhatsApp / ligação) — não existe coluna

## O que não vai ser construído

O diálogo cliente↔vendedora. Registrado aqui para não ser reproposto.

Ver [[2026-08-24]] e [[canal-interno-vendedora]].
