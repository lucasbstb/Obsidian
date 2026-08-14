# A.T Jewel

> _(descreva aqui o sistema em 2–3 linhas, no mesmo tom do MktAction)_

---

## Repositórios

|Repo|O que é|
|---|---|
|`at-jewel-back`|API (NestJS)|
|`at-jewel-front`|Aplicação (Next)|

## Startar

```bash
# Na pasta at-jewel-back
docker compose up -d postgres   # Cria e starta o container do banco local
docker compose start            # Apenas starta o container já criado

npm run start:dev               # Starta a API local

# Na pasta at-jewel-front
npm run dev                     # Starta o front local
```

## Shutdown

```bash
CTRL + C                # Para de rodar back e front

# Na pasta at-jewel-back
docker compose down     # Remove o container da lista (mantém o volume do banco)
docker compose stop     # Para o container, mantém na lista e mantém o banco
```

## Portas

|Serviço|URL|
|---|---|
|Front (Next)|http://localhost:3001|
|Backend (Nest)|http://localhost:3000|
|Postgres|localhost:5432|

## Credenciais locais

Só valem no ambiente de desenvolvimento.

| Login                           | Senha        | Perfil |
| ------------------------------- | ------------ | ------ |
| l.barbosa@stbtecnologias.com.br | stb@lucas123 | Admin  |


## Branches

`main` (produção) ← PR ← `dev` (integração) ← PR ← branch do dev (ex.: `lucasdev`)

Mesmo fluxo nos dois repositórios (`at-jewel-back` e `at-jewel-front`).

