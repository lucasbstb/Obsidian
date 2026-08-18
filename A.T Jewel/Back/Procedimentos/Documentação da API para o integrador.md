# Documentação da API para o integrador

> Procedimento vivo — atualizar quando a API mudar, sem criar versão nova.
> Relacionados: [[2026-08-14]], [[Aplicar migração de banco]],
> [[2026-08-11 — Integração ERP Safira — Levantamento de Requisitos]].

Criada em 14/08/2026, para substituir a coleção do Postman —
`AT-JEWEL-API.postman_collection.json` — que o Lucas considerou confusa demais
para entregar a alguém de fora.

---

## Onde está

**https://claude.ai/code/artifact/ecfc3220-b19e-48c7-bbd5-9148dfb32980**

Página HTML autocontida. Abre em qualquer navegador, sem instalar nada, e vira
PDF com **Ctrl+P** — o índice lateral some na impressão e os blocos de endpoint
não quebram no meio da página.

Nasceu **privada**. Só fica visível para outras pessoas pelo menu de
compartilhar da própria página.

---

## O que ela cobre

Só os cadastros que a integração usa. Não é documentação da API inteira — são
117 rotas em 21 controllers, e documentar tudo produziria algo tão confuso
quanto a coleção que ela substitui.

```
clientes           vendedoras         fornecedores
empresas           formas-pagamento
grupos-estoque     locais-estoque     estoque          <- 18/08
```

De cada recurso: as rotas com o escopo exigido, os campos que podem ser enviados
com tipo, obrigatoriedade e limite, os filtros de listagem, um exemplo de
resposta e a lista fechada de valores aceitos nos campos de enum.

Tudo extraído dos DTOs e das entidades, não escrito de memória.

### Revisão de 18/08/2026

Entraram os três recursos de estoque e, em TODOS os recursos, o campo `idErp`.

A seção nova que mais importa e a de "Os dois identificadores": o integrador
precisa entender que `idErp` e identidade e `codigoErp` e atributo. Se ele
sincronizar pelo codigo e alguem renomear na loja, criamos registro duplicado —
e ninguem percebe, porque os dois seguem sendo atualizados.

Tambem ficou escrito, com destaque, que **quantidade negativa e valida**. Sem
isso alguem do outro lado "corrige" o sinal antes de enviar e a contrapartida da
consignacao se perde.

O arquivo local e `Documents/API A.T. Jewel - Cadastros.md` — 498 para 710
linhas. O artifact ainda NAO foi regerado com essa revisao.

---

## Duas coisas que criam obrigação

**1. Ela carrega uma chave de API.** Decisão do Lucas, contra a recomendação de
manter a chave fora do documento. Consequências:

- **O link virou segredo.** Quem tem o link tem a chave — precisa do mesmo
  cuidado: não colar em grupo, não encaminhar adiante.
- **A página tem prazo.** Quando a chave for revogada ou rotacionada, ela fica
  errada. Atualizar as duas coisas no mesmo movimento.

**2. Ela envelhece como qualquer documento gerado do código.** É exatamente o
erro que a `DOCUMENTACAO TECNICA DO PROJETO.MD` cometeu: lista escopos que não
existem mais e por isso deixou de ser confiável.

**Quando revisar:** rota nova ou removida nos cinco cadastros; campo novo em
DTO; valor novo em enum; mudança de escopo exigido; chave rotacionada.

---

## Decisões de conteúdo que valem manter

Registradas porque foram discutidas e a razão não é óbvia:

- **Só o que pode ser enviado.** Campo que a integração não deve preencher
  (`adminUserId`) saiu das tabelas de envio — listar para dizer "não envie" é
  convite ao erro. Ele aparece só no exemplo de resposta, com uma linha de
  explicação.
- **Campos gerados pelo CRM** — `id`, `criadoEm`, `atualizadoEm` — declarados
  como tal nas Convenções. `criadoEm` é o instante em que o registro entrou no
  **nosso** banco, não a data de cadastro no sistema de origem.
- **Nada de rota que não está documentada.** A primeira versão explicava o `403`
  citando `/vendas`, que não faz parte deste recorte — levantava mais dúvida do
  que resolvia.
- **Escrita do ponto de vista de quem recebe a chave**, não de quem a gera. O
  integrador não tem acesso ao painel.

---

## Uma chave, não cinco

Discussão de 14/08. O Lucas havia gerado cinco chaves de teste, uma por recurso.

**Separar chave só protege se as chaves moram em lugares diferentes.** As cinco
iriam para o mesmo integrador, na mesma configuração, no mesmo servidor — vazou
uma, vazaram as cinco. Paga-se o custo da separação sem receber o benefício.

O critério certo é **por sistema**:

```
integracao-safira     os 6 escopos de cadastro    a Conexa
n8n-anastasia         clientes:read/write         os agentes
integracao-catalogo   produtos:read/write         o catálogo
```

Aí o isolamento é real, porque são máquinas e responsabilidades diferentes.
