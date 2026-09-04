# Documento de Melhorias v1.0 — status

Situação em **04/09/2026**. `T` feito · `—` pendente

**21 de 26 fechados.** Os 5 que sobram são a mesma frente.

---

## Feitos

| | Item | 03/09 | agora | como |
|---|---|---|---|---|
| MEL-01 | ID do produto no cadastro | N | **T** | modal de Nova peça, com o botão antes do Exportar |
| MEL-02 | Miniatura e coluna de ID | N | **T** | miniatura com proxy no back, e coluna de Código com clique para copiar |
| MEL-03 | Perfis Estoquista e Marketing | P | **T** | Contemplado no sistema |
| MEL-04 | Juros manuais sobre preço | P | **T** | sem juro informado não há juro; parcelas e juro editáveis no card |
| MEL-05 | Textos em tooltip | N | **T** | componente `Dica`; cinco parágrafos viraram ⓘ |
| MEL-06 | Aprovação pelo marketing | P/C | **T** | Contemplado no sistema |
| MEL-07 | Observação individual por imagem | P | **T** | coluna na referência, lista vertical, e a nota viaja na exportação |
| MEL-08 | Cabeçalho padronizado / capa | P | **T** | capa escolhida entre as referências, aparecendo no card da lista |
| MEL-09 | Exportação em alta qualidade | P | **T** | decisão: já é o melhor que temos; o teto é do modelo |
| MEL-10 | PDF como referência | N | **T** | aceita PDF até 100 MB, com cartão de arquivo no lugar da miniatura |
| MEL-11 | Destaque do botão Anexar | N | **T** | sólido e dourado, no lugar do tracejado apagado |
| MEL-12 | "ChatGpt" → "Agente" | N | **T** | e as duas frases que mentiam sobre o mecanismo foram corrigidas |
| MEL-18 | Papéis de Helena e Anastácia | T | **T** | já existia |
| MEL-19 | Módulo de Consignações | T | **T** | já existia |
| MEL-20 | Módulo de Ocorrências | P | **T** | cliente opcional, fotos por episódio, e filtro por cliente |
| MEL-21 | Lista de clientes anonimizada | P | **T** | a tabela de clientes; o recorte por carteira é o que anonimiza |
| MEL-22 | Permissões granulares por perfil | T | **T** | já existia |
| MEL-23 | Proteção de dados de terceiros | P | **T** | `clientes:read_all` e `EscopoClientesService` |
| MEL-24 | Mensagens com opções em lista | C | **T** | deixou de ser conflito — lista vale no canal interno |
| MEL-25 | API de vendas para o Alessandro | P | **T** | feito e entregue |
| MEL-26 | Limpeza da tela de Vendas | ? | **T** | os 1.269 registros de seed apagados em 03/09 |

---

## Pendentes

### Travados numa pergunta só

| | Item | hoje | o que falta |
|---|---|---|---|
| MEL-13 | Aba única, eliminando Demandas | P/C | decidido: desativar sem excluir. A execução depende do MEL-14 |
| MEL-14 | Timeline por vendedora | P | existe em `/admin/auditoria`, mas plota episódios e relatos, não conversas |
| MEL-15 | Regra de encerramento | P | o desfecho vem de LER a conversa, não de digitar à mão |
| MEL-16 | Abrir e consultar conversas | N | o cliente do WAHA só tem a prévia da última mensagem |
| MEL-17 | Captura em celular corporativo | P | a migração 39 criou as colunas e diz que não liga ao WAHA |
