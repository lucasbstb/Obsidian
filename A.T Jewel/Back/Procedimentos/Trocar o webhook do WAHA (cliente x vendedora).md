# Trocar o webhook do WAHA (cliente × vendedora)

> Escrito em 20/08/2026, no dia em que o webhook foi apontado para o back pela
> primeira vez. Enquanto existir **um número só**, ele serve ou o cliente ou a
> vendedora — nunca os dois. Este roteiro é para virar a chave nos dois sentidos
> sem quebrar nada.

**Onde o webhook mora:** na **configuração da sessão dentro do WAHA**. Não é
arquivo do nosso código, não está em `.env` nosso, não adianta procurar no
repositório. É estado do próprio WAHA, e some do radar por isso.

**As duas máquinas** — e é o erro mais fácil de cometer:

```
10.29.0.151   WAHA e atwpp        (EVO-N8N)
10.29.0.137   front, back e Postgres  (AT-AGENTE)
```

Em 20/08 eu quase mandei o webhook para `10.29.0.151:3000` achando que era tudo
o mesmo servidor. Não é. **Confira antes**, com `hostname -I` no AT-AGENTE.

---

## Para onde apontar

| Destino | URL | Quem é atendido |
|---|---|---|
| **atwpp** | `http://10.29.0.151:8099/whatsapp/webhook` | cliente — triagem da Anastasia |
| **back** | `http://10.29.0.137:3000/whatsapp/webhook` | vendedora — canal interno |

**Só um por vez.** Apontar para o back deixa a triagem do cliente **muda** —
quem escrever para a loja não recebe nada, nem mensagem automática. Apontar para
o atwpp faz a vendedora receber triagem de cliente como resposta.

Quando existirem dois números, cada sessão tem o seu webhook e essa escolha
deixa de existir.

---

## Como trocar

Dá para fazer pelo Thunder Client, pelo Insomnia ou por `curl`. O painel do WAHA
mostra as sessões mas **não deixa editar o webhook**.

### 1. Ver o que está lá hoje

```
GET  https://waha.stbdesenvolvimento.com/api/sessions/default
Header:  X-Api-Key: <WAHA_API_KEY do .env do back>
```

O que interessa está em `config.webhooks`:

```json
"config": {
  "webhooks": [
    {
      "url": "http://10.29.0.151:8099/whatsapp/webhook",
      "events": ["message"],
      "customHeaders": [
        { "name": "X-Webhook-Token", "value": "054a46..." }
      ]
    }
  ]
}
```

### 2. Mandar de volta com a URL trocada

```
PUT  https://waha.stbdesenvolvimento.com/api/sessions/default
Headers:  X-Api-Key: <a mesma>
          Content-Type: application/json
```

**Copie o bloco `config` inteiro da resposta do GET**, cole no corpo, e mude
**só a URL**. Assim você não digita o token em lugar nenhum.

> **O `PUT` substitui, não edita.** Se o `customHeaders` não for junto, ele deixa
> de existir — e aí o back recusa tudo com **401**. O sintoma parece "não está
> chegando nada", e você vai caçar rede quando o problema é o cabeçalho.

### 3. Conferir que a sessão voltou

```
GET  https://waha.stbdesenvolvimento.com/api/sessions/default
```

Tem que voltar `"status": "WORKING"` e a URL nova.

O WAHA **reinicia a sessão** para aplicar a configuração — são alguns segundos
fora do ar. A autenticação fica guardada: **não pede QR de novo**. QR só é pedido
depois de `logout`, `delete`, ou se alguém desconectar o aparelho pelo celular.
Então: **não use nada com `logout` ou `delete` no nome.**

Se mesmo assim voltar em `SCAN_QR_CODE`, a tela `/admin/whatsapp` do painel
mostra o QR — ver [[Reconectar o WhatsApp da Anastasia (WAHA)]].

### 4. Testar

Mande qualquer mensagem para o número e, no AT-AGENTE:

```bash
docker logs --tail 300 atjewel_api 2>&1 | grep -vE "^query:" | tail -30
```

O `grep -v` não é frescura: o TypeORM está com `logging: true` em produção e o
log é ilegível sem ele (ver [[back-producao-roda-como-development]]).

| O que aparecer | Significa |
|---|---|
| `remetente nao reconhecido — ignorada` | **funcionou** — chegou no back, e ele te ignorou porque seu número não é de vendedora |
| `401` / `X-Webhook-Token inválido` | o cabeçalho não foi junto no PUT |
| nada | o WAHA não alcança a porta — firewall entre `.151` e `.137` |

---

## O passo que anda junto, e é fácil esquecer

Existe uma **vendedora de teste** em produção, criada em 20/08:

```
nome        Lucas Barbosa(DEV-TESTE)
codigo_erp  TESTE-VD
status      DISPONIVEL
```

Enquanto o webhook aponta para o back, ela é inofensiva — não há triagem
rodando. **Ao devolver o webhook para o atwpp, a Anastasia volta a rotear
clientes e ela entra no sorteio.** Desative antes:

```bash
curl -X PATCH http://10.29.0.137:3000/vendedoras/<id> \
  -H 'content-type: application/json' -H 'x-api-key: <chave>' \
  -d '{"statusDisponibilidade":"AUSENTE"}'
```

Existe também um cliente de teste, `TESTE-CLI` / "Ana Livia(Cliente Teste -
DEV)", vinculado a ela.

---

## Uma coisa que ficou por verificar

Se o webhook estiver **fixado por variável de ambiente** no contêiner do WAHA
(`WHATSAPP_HOOK_URL` ou parecida), a troca pela API funciona agora e **volta
sozinha no próximo restart do WAHA**. Isso não foi confirmado. Para checar, no
host `10.29.0.151`:

```bash
docker inspect waha --format '{{range .Config.Env}}{{println .}}{{end}}' | grep -i hook
```

Se voltar alguma linha com `HOOK`, a mudança tem que ser no `docker-compose` do
WAHA, não pela API.
