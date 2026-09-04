# 2026-09-04 — Atendimento no WhatsApp — Desenho

Desenho pedido pelo Lucas em 04/09, guardado para ser executado depois. **Nenhuma
linha de código foi escrita.** Cobre quatro itens do Documento de Melhorias de uma
vez: **MEL-13**, **MEL-14**, **MEL-16** e **MEL-17**.

---

## O que ele pediu, nas palavras dele

> "O MEL-13 é assim: a tela de demandas você desativa, mas não exclui ainda. E
> essa tela vai virar a MEL-14, que vai ser assim:
>
> Na aba de WhatsApp vai ter duas abas. Uma que já tem, que seria o WhatsApp, mas
> vai ter que acrescentar todas as vendedoras. Porque ela tem 2 celulares: o
> interno e o externo (pessoal dela). No interno, que seria o corporativo, vamos
> ter que ler os QR Code delas e ter acesso às conversas. No pessoal seria aquele
> fluxo do WhatsApp, onde você manda perguntas, tipo 'como foi o atendimento da
> Maria?', aí ela anota e segue o fluxo.
>
> E a outra aba seria a timeline, que você iria alimentar com as coisas que a
> vendedora fez: 8 horas agendou um cliente, 9 horas fez tal coisa.
>
> Ao clicar em uma vendedora abriria as conversas, e também poderia clicar nas
> conversas para ler cada uma delas — você, agente, também."

Ele mandou um **modelo de tela** junto: lista de Conexões com "Loja (Anastasia)"
no topo e uma linha por vendedora, cada uma com estado (Conectado / Falhou /
Desconectado), o número, quando sincronizou, quantos chats ativos, e os botões
Atualizar / Desconectar / Conectar.

---

## O ponto que trava tudo: os nomes estão invertidos

A migração 39 nomeia as colunas **por interlocutor**:

```
whatsapp_interno  ->  o canal com a IA      (Elena, Anastasia)  = celular PESSOAL
whatsapp_externo  ->  o canal com o CLIENTE                     = chip CORPORATIVO
```

O Lucas nomeia **por posse**: interno = da empresa, externo = dela.

**São exatamente opostos.** Construir pelas palavras dele sem resolver isso faria
o sistema pedir o QR Code do **celular pessoal** da vendedora — e o QR do WAHA dá
acesso à **conta inteira**: toda conversa daquele número, a mídia decifrada e o
poder de enviar. Seria a vida particular dela dentro do painel.

**Pergunta feita a ele em 04/09, sem resposta ainda:** o número que recebe o QR
Code é o chip que a empresa entregou — o que aparece para o cliente — certo?

Se sim, no banco ele é o `whatsapp_externo`, e a nomenclatura precisa ser
renomeada ou documentada com muita ênfase, porque a confusão vai voltar toda vez
que alguém novo ler as duas colunas.

---

## O que existe hoje

| | |
|---|---|
| **A tela de Conexões** | O modelo que ele mandou é uma maquete. Hoje o painel tem **uma conexão só** |
| **O gateway** | `WAHA_SESSION` é uma **variável de ambiente única**, usada em toda chamada do `waha.gateway.ts`. Não há multi-sessão |
| **Ler conversa** | Não existe. O cliente do WAHA só tem `chats/overview`, a prévia da última mensagem. Sem endpoint de mensagens |
| **A timeline** | **Já existe**, em `/admin/auditoria`: cartão por vendedora, linha do tempo com hora, etapa, cliente, relato e próximo contato. Plota **episódios e relatos**, não conversas |
| **Demandas** | Fila de chamado interno para a equipe técnica. Desativar é tirar do menu; os dados e as rotas ficam |
| **Migração 39** | Criou `whatsapp_externo` e o hash, e **diz na própria cabeça** que não liga ao WAHA, não roteia e não captura. Zero código a usa |

---

## A ordem proposta

1. **Demandas fora do menu.** Meia hora, sem risco. Rota e dados intactos.

2. **Sessão por vendedora no back.** A sessão vira **parâmetro** em todo o
   gateway (hoje é lida do ambiente em cinco lugares), uma tabela liga vendedora →
   sessão, e o webhook passa a saber de qual sessão veio cada mensagem —
   hoje o `RotearMensagemInternaUseCase` identifica a vendedora **unicamente pelo
   `whatsapp_interno_hash`**.
   **É a fundação. Sem ela, nada do resto existe.**

3. **A tela de Conexões com todas as vendedoras.** QR por vendedora, no desenho
   da maquete.

4. **Ler as conversas.** Endpoint de mensagens no cliente do WAHA, tela de
   leitura, e o acesso da agente ao conteúdo.
   **É o passo que muda o que o painel expõe sobre pessoas** — ver abaixo.

5. **A aba Timeline.** Mover a auditoria para dentro do WhatsApp e passar a
   alimentá-la também com o que vier das conversas capturadas.

---

## A consequência que não pode passar despercebida

Conectar o número corporativo ao WAHA torna **legível pelo painel toda conversa
daquele número** — inclusive a conversa cliente–vendedora, que o sistema hoje
assume ser invisível. Está escrito na migração 39 e no planejamento de 25/08.

Para um chip corporativo isso é o objetivo, não um efeito colateral. Mas:

- muda o que a empresa sabe sobre o atendimento, e isso deveria ser dito às
  vendedoras antes de o QR ser lido;
- muda o que um vazamento do painel expõe;
- e o passo 4 dá esse conteúdo **também à agente**, o que significa que trechos de
  conversa com cliente passam a viajar para o provedor do modelo.

Nada disso impede o trabalho. Mas é decisão consciente, não detalhe técnico.

---

## Documentos relacionados

- [[2026-09-03 — Documento de Melhorias v1.0 — Batimento]] — onde MEL-13 a MEL-18
  foram classificados
- Planejamento de 25/08/2026 — a análise original de conectar o corporativo ao
  WAHA
