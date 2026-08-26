# 2026-08-25 — WhatsApp corporativo da vendedora no WAHA

> **Nada disto foi construído.** O que existe hoje é só a coluna
> `whatsapp_externo` (migração 39). Este documento é o desenho, para a decisão
> ser tomada com o preço na mesa.

---

## O que muda

Cada vendedora passa a ter **dois** números, e a diferença entre eles não é de
posse — é de **interlocutor**:

| coluna | fala com | de quem é o chip |
|---|---|---|
| `whatsapp_interno` | a IA (Elena, Anastasia) | pessoal dela |
| `whatsapp_externo` | o **cliente** | corporativo, da empresa |

Duas decisões já tomadas (25/08, com o Lucas):

1. A vendedora poderá escrever para a Elena **dos dois números**.
2. O corporativo **será conectado ao WAHA** — é essa a intenção.

A segunda é a que carrega o peso todo.

---

## A premissa que cai

Está escrito no `ConsultarAuditoriaUseCase`, e é o motivo de a tela de auditoria
ter o recorte que tem:

> *"O QUE ESTA LEITURA NAO E: nao e a conversa entre cliente e vendedora. Essa
> acontece entre dois telefones pessoais, fora de qualquer sistema nosso, e
> captura-la exigiria conectar o WhatsApp pessoal de gente externa."*

Com o corporativo no WAHA, **isso deixa de ser verdade**. Não é detalhe de
implementação: é a fronteira que define o que o produto é.

### O que se ganha

- A auditoria deixa de depender do **relato** da vendedora e passa a ver o que
  aconteceu. Hoje toda a tela de auditoria repousa sobre ela ter respondido a
  cobrança.
- `primeiro_contato_em` vira **automático** — hoje é a pendência #4, e o
  relógio do SLA de primeiro contato não para sem ele.
- O vínculo atendimento↔venda ganha evidência: dá para ver a negociação, e não
  só o desfecho declarado.

### O que se aceita junto

- **Conversa de cliente passa a ser armazenada.** Isso tem base legal, prazo de
  retenção e direito de acesso — não é decisão de engenharia.
- **A vendedora precisa saber**, com todas as letras, que aquele número é lido.
  Descobrir depois destrói a confiança do canal interno inteiro.
- **O WAHA lê a CONTA, não a loja.** O QR dá acesso a *toda* conversa daquele
  número, à mídia decifrada e ao poder de enviar. Se o chip corporativo estiver
  no aparelho pessoal dela e ela usar para qualquer outra coisa, isso também
  entra. Ver [[waha-le-a-conta-inteira]].

---

## O problema central: hoje o sistema ignora a sessão

Esta é a parte que precisa ser resolvida antes de qualquer coisa, e ela não é
óbvia olhando o código de fora.

**Na entrada.** O payload do WAHA traz `session`, e o `extrairMensagemRecebida`
declara o campo — e **descarta**. O roteamento decide tudo pelo **remetente**:

```
telefone de vendedora ativa   ->  Elena
telefone de usuário de gestão ->  Anastasia
qualquer outro                ->  silêncio
```

**Na saída.** O `WahaGateway` lê `WAHA_SESSION` da env em toda chamada —
`resolverChatId`, `resolverRemetente`, `enviarTexto`. **Uma sessão fixa, para
tudo.**

Isso funciona enquanto existe **uma** conta. Com o corporativo da Camila como
segunda sessão, a mesma pessoa — um cliente — pode estar falando com a **loja**
ou com a **Camila**, e o tratamento tem que ser diferente. O remetente é
idêntico nos dois casos. **Quem distingue é a sessão**, isto é, o
destinatário.

> Hoje o sistema pergunta *"quem está falando?"*. Com N sessões ele precisa
> perguntar também *"com quem essa pessoa está falando?"* — e essa segunda
> pergunta não existe em lugar nenhum do código.

### O risco concreto, se isso for ignorado

Uma sessão nova apontada para o mesmo webhook, sem tratar `session`:

- cliente da Camila escreve para ela → o roteador não reconhece o remetente
  (não é vendedora nem gestão) → **silêncio**. Salvo por acaso, pelo
  default-deny.
- mas qualquer agente que responda cliente pelo número da loja passa a poder
  responder **no lugar da vendedora**, porque nada no caminho sabe que aquela
  mensagem não era para a loja.
- e uma resposta enviada sairia pela `WAHA_SESSION` da env — ou seja, **pelo
  número da loja**, para um cliente que escreveu para a Camila.

O default-deny segura o primeiro caso. Ele não segura os outros dois.

---

## Desenho proposto

### 1. A sessão vira dado de primeira classe

- `extrairMensagemRecebida` passa a **propagar** `session`.
- `IWhatsappGateway` recebe a sessão por **parâmetro**, não da env. A
  `WAHA_SESSION` continua como padrão para a conta da loja.
- Uma tabela `whatsapp_sessoes` mapeia **sessão → dono**:

```
sessao (PK)   |  tipo            |  vendedora_id
--------------+------------------+---------------
default       |  LOJA            |  null
vd-camila     |  VENDEDORA       |  <uuid>
```

Sem esse mapa, a única forma de saber de quem é a sessão seria por convenção de
nome — que quebra silenciosamente no dia em que alguém renomear.

### 2. O roteador passa a ter duas perguntas

```
     ┌── sessão da LOJA ────────► quem escreve?
     │                             vendedora → Elena
     │                             gestão    → Anastasia
     │                             cliente   → triagem
     │
     └── sessão de VENDEDORA ───► quem escreve?
                                   a própria (fromMe) → registrar, não responder
                                   cliente            → registrar, NÃO responder
```

**Ninguém responde no canal da vendedora.** Esta é a regra que eu defenderia com
mais força: o corporativo é dela, e a IA entrando ali por conta própria
transforma um canal de trabalho num canal de vigilância que também fala. Se um
dia houver resposta automática nesse canal, que seja decisão explícita, com a
vendedora sabendo — não consequência de arquitetura.

### 3. O registro

A conversa capturada vira interação do atendimento em curso daquele cliente com
aquela vendedora — a linha do tempo já existe, e é onde a auditoria lê. O que
falta é a origem: distinguir *"a vendedora relatou"* de *"o sistema viu"*, do
mesmo jeito que a tela já separa automático de gente.

---

## O que isso custa em infra

| | |
|---|---|
| sessões | 1 hoje → **1 + uma por vendedora** (8 hoje) |
| memória | cada sessão do WAHA é uma instância de navegador — este é o número que falta |
| QR | um por vendedora, presencial, e **de novo a cada queda de sessão** |
| reconexão | sessão cai sozinha; sem alguém olhando, o número fica mudo sem avisar |

**O número de memória por sessão é o dado que decide se isso cabe na VPS
atual.** Ele está na mesma lista dos quatro que faltam para o documento do
DevOps (`docker stats`, tamanho do banco, `df -h`, `free -h`).

E há uma consequência operacional que não é técnica: **cada QR é uma ida até a
vendedora.** Oito vendedoras, e a sessão cai sozinha de tempos em tempos.

---

## Fases

| fase | entrega | depende de |
|---|---|---|
| **0** | coluna `whatsapp_externo` — **feito**, migração 39 | — |
| **1** | tela para preencher os dois números | nada |
| **2** | a Elena reconhece a vendedora pelos **dois** números | fase 1 |
| **3** | sessão como dado: propagar no webhook, parametrizar no gateway, tabela `whatsapp_sessoes` | — |
| **4** | uma sessão corporativa, **uma vendedora só**, registrando e sem responder | fase 3 |
| **5** | avaliar com dado real: memória, quedas, o que a captura acrescenta à auditoria | fase 4 |
| **6** | o resto da equipe | fase 5 |

A fase 4 com **uma** vendedora não é timidez. É o único jeito de medir o custo
de sessão e a taxa de queda sem descobrir com oito números mudos ao mesmo
tempo — e não existe homolog para ensaiar
([[producao-roda-next-dev-bind-mount]]).

---

## Decisões que ainda não foram tomadas

1. **A conversa capturada fica visível para quem?** Gestão vê a conversa
   inteira, ou só o fato de que houve contato? A regra vigente é *"apenas o
   ADM tem acesso"* — mas conversa é outra ordem de grandeza em relação a
   feedback.
2. **Retenção.** Conversa de cliente guardada para sempre é passivo. Trinta,
   noventa, cento e oitenta dias?
3. **O cliente é avisado?** A mensagem-padrão de canal corporativo gravado é
   prática comum, e é o que sustenta a base legal.
4. **Se a vendedora sai da empresa**, a sessão dela é derrubada e o número
   reciclado — o `UNIQUE` do hash exige limpar o cadastro antigo antes. Isso
   pede um procedimento, não só código.
5. **Fase 2 sozinha muda a superfície do canal interno.** Reconhecer os dois
   números significa que quem tiver o corporativo em mãos fala com a Elena
   como se fosse ela. Chip corporativo circula mais que celular pessoal.

---

## O que já está pronto e serve

- `whatsapp_externo` + `whatsapp_externo_hash` (UNIQUE, cifrado, indexado) —
  migração 39
- `buscarPorWhatsappHash` e `variantesTelefone`, que resolvem o problema do
  nono dígito
- `resolverRemetente`, que já traduz LID → telefone na borda
- o `session` chegando no payload — só falta não ser descartado

## O que a migração 39 deliberadamente **não** faz

Não liga nada ao WAHA, não muda roteamento, não captura conversa. O
`RotearMensagemInternaUseCase` continua reconhecendo a vendedora **só** pelo
`whatsapp_interno_hash`.

Uma limitação registrada no próprio arquivo: há `UNIQUE` por **coluna**, não no
**conjunto** das duas. Nada impede que o corporativo da Camila seja igual ao
pessoal da Beatriz. Postgres não expressa isso com `UNIQUE` — precisaria de uma
tabela `vendedora_whatsapps` (uma linha por número, `UNIQUE` no hash), que é a
modelagem correta e uma remodelagem que não se justifica com dois números fixos
por pessoa. **Até lá a trava fica na aplicação — e a fase 2 não pode ser feita
sem estendê-la aos dois campos**, senão a colisão vira ambiguidade de
identidade no canal.

---

Ver [[2026-08-25]] para o dia, e [[waha-le-a-conta-inteira]] para o que uma
sessão do WAHA realmente enxerga.
