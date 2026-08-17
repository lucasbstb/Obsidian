# Reconectar o WhatsApp da Anastasia (WAHA)

> Escrito em 17/08/2026, depois de a sessão ficar fora do ar por cerca de duas
> semanas sem ninguém perceber. O diagnóstico levou quase duas horas porque os
> sintomas apontavam para o lado errado. Com este roteiro, leva dez minutos.

**Onde tudo isso roda:** host `10.29.0.151` (EVO-N8N), container `waha`, porta
`8145`. Não é a máquina do CRM — front e back ficam em outro servidor.

**Qual número está pareado hoje:** `558535141045` ("Artz Teste"), um número de
**testes**, de outra empresa da casa. Por isso a queda de agosto não teve custo
comercial — o que apareceu no lugar foi conversa interna, não cliente. O número
definitivo da A.T. Jewel entra depois, e **a partir daí a mesma falha significa
loja muda**. Este roteiro existe para esse dia.

---

## Sintoma

- A tela `/admin/whatsapp` mostra *"Falha ao consultar o status do WhatsApp"*,
  ou fica em *"Aguardando leitura do QR"* para sempre
- Lê-se o QR e **nada acontece** — nenhum erro, simplesmente não conecta
- Quem escreve para o número não recebe resposta da Anastasia
- O webhook nunca dispara, então **nada é registrado**: pelo CRM parece só um
  dia sem movimento. É por isso que a queda passa despercebida

---

## Diagnóstico

Tudo daqui para baixo roda no console do `10.29.0.151`. Primeiro a chave:

```
KEY=$(docker inspect waha --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^WAHA_API_KEY=' | cut -d= -f2)
```

Depois o estado da sessão:

```
curl -s -H "X-Api-Key: $KEY" http://localhost:8145/api/sessions/default
```

O que olhar na resposta:

| Campo | Significado |
|---|---|
| `status` | `WORKING` = ok · `SCAN_QR_CODE` = esperando pareamento · `FAILED` = quebrada |
| `engine.gows.connected` | **o que mais importa.** `false` = sem conexão com o WhatsApp |
| `engine.grpc.client` | conversa interna WAHA↔GOWS. `READY` mesmo com tudo quebrado |

**`connected: false` é o sinal.** Enquanto ele for `false`, ler QR é perda de
tempo: não existe canal com o WhatsApp para o pareamento acontecer.

### A linha que dá o veredito

```
docker logs --since 10m waha 2>&1 | grep -i "gows\|websocket\|logged\|401\|error"
```

Procurar por:

```
[Session/default/Client] Got 401: logged out from another device
connect failure, sending LoggedOut event and deleting session
```

Isso quer dizer: **alguém removeu o WAHA em "Aparelhos conectados" no celular**,
ou o WhatsApp expirou a vinculação. O WAHA fica tentando reconectar com a
credencial de um aparelho que não existe mais, leva 401, e apaga a sessão.

---

## O que NÃO resolve (e custou tempo)

| Tentativa | Por que falha |
|---|---|
| Ler o QR de novo | sem websocket não há pareamento — o QR é gerado localmente |
| `docker restart waha` | sobe e volta a `FAILED` na hora: o problema é a credencial |
| `POST /auth/request-code` | responde 500 `websocket not connected` |
| Trocar a imagem do WAHA | a versão está correta; ela só estava relatando a verdade |

**A pista falsa:** o erro visível é `websocket not connected`, que parece
problema de rede. Não é — é **consequência** do 401. A saída do container está
liberada, e dá para provar:

```
docker exec waha node -e "fetch('https://web.whatsapp.com').then(r=>console.log('saida ok',r.status)).catch(e=>console.log('BLOQUEADA',e.message))"
```

---

## O procedimento

**Com o celular do número em mãos.** O pareamento é presencial.

```
KEY=$(docker inspect waha --format '{{range .Config.Env}}{{println .}}{{end}}' | grep '^WAHA_API_KEY=' | cut -d= -f2)

curl -s -X POST -H "X-Api-Key: $KEY" http://localhost:8145/api/sessions/default/logout
curl -s -X POST -H "X-Api-Key: $KEY" http://localhost:8145/api/sessions/default/start
sleep 8
curl -s -H "X-Api-Key: $KEY" http://localhost:8145/api/sessions/default | grep -o '"status":"[^"]*"'
```

O `logout` é o passo que resolve: limpa a credencial do aparelho removido. Sem
ele a sessão renasce com o mesmo resíduo e leva 401 de novo.

**Conferir antes de seguir:** o status precisa estar `SCAN_QR_CODE` **e**
`gows.connected` precisa ter virado `true`. Se `connected` ainda for `false`,
não adianta ir para o QR.

### Parear — dois caminhos

**Pelo QR**, em `/admin/whatsapp` (local ou produção, tanto faz — as duas telas
falam com o mesmo WAHA). Recarregar a página para gerar um QR novo, com o
celular **já aberto** em *Aparelhos conectados → Conectar um aparelho*. O QR do
WhatsApp vive uns 20 segundos; o erro clássico é procurar o menu com o QR já na
tela.

**Pelo código de 8 dígitos**, mais tranquilo porque dura minutos:

```
curl -s -X POST -H "X-Api-Key: $KEY" -H "Content-Type: application/json" \
  -d '{"phoneNumber":"558535141045"}' \
  http://localhost:8145/api/default/auth/request-code
```

No celular: *Aparelhos conectados → Conectar um aparelho → Conectar com número
de telefone* → digitar o código.

> Este endpoint só funciona **depois** do `logout`+`start`. Antes disso ele
> responde 500 `websocket not connected`.

---

## Verificação

```
curl -s -H "X-Api-Key: $KEY" http://localhost:8145/api/sessions/default | grep -o '"status":"[^"]*"'
```

`WORKING` é o alvo. Na tela: **Conectado**, com o número e o nome do aparelho.

A lista de chats demora um pouco: o WhatsApp entrega o histórico aos poucos
depois que a sessão sobe. Ver 0 chats logo após conectar é normal — *Atualizar
chats* daí a alguns segundos.

---

## Como evitar a próxima

**A causa é humana:** alguém removeu o aparelho conectado no celular. Vale
avisar quem tem acesso ao aparelho que aquele dispositivo não pode ser removido.

**E ninguém percebeu que caiu.** O número ficou mudo por cerca de duas semanas
e a falha só apareceu porque fomos mexer em outra coisa. Não existe alerta: o
CRM não checa o status da sessão, e o silêncio de um canal é indistinguível de
um dia parado.

Desta vez o custo foi zero, porque o número é de teste. Quando o número real
entrar, duas semanas assim são duas semanas de cliente escrevendo para o vazio
— e continuaria invisível pelo mesmo motivo. Um verificador que consulte
`gows.connected` de tempos em tempos e avise quando sair de `WORKING` resolve, e
é exatamente o papel da Sofia, registrado em
[[2026-08-17 — Agente no WhatsApp para Vendedoras]].

---

## Contexto útil

- **Sessão = número.** Cada número pareado é uma sessão do WAHA. A `default` é a
  da Anastasia. A Elena terá a sua, e as duas são independentes: uma pode estar
  quebrada e a outra funcionando.
- **O pareamento não vive no container.** Fica em Postgres
  (`WHATSAPP_SESSIONS_POSTGRESQL_URL`), então recriar o container não perde a
  sessão — e por isso mesmo o restart não conserta este problema.
- **A tela do CRM é um proxy**: `/admin/whatsapp` → `at-jewel-back`
  (`whatsapp-admin.controller.ts`, JWT + ADMIN) → `waha-admin.client.ts` → WAHA.
  A API key do WAHA nunca chega ao navegador.
- **A imagem é `devlikeapro/waha-plus`, privada no Docker Hub.** Ninguém sabe
  quem tem a credencial — então este container **não pode ser atualizado nem
  recriado em outro host** hoje. Se um dia precisar, a saída é migrar para
  `devlikeapro/waha` (Core), que é público e desde a `2026.6.1` tem tudo o que
  era do Plus, sessões ilimitadas incluídas. A instância roda a `2026.5.1`.
