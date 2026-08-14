# MktAction

SaaS B2B de marketing de campo eleitoral: planeja, executa e mede ações de rua
(panfletagem, carreata, caminhada) com geolocalização, confirmação de presença
por GPS e leitura de resultados.

---

## Startar

```bash
# Rodar na pasta do projeto
npx supabase start            # Starta o container do banco local(Falhando)
npm run supabase:start        # Starta o container do banco local
npm run dev                   # Starta o projeto local
```

## Shutdown

```bash
npx supabase stop           # Para os containers, mantém o banco(Falhando)
npm run supabase:stop       # Starta o container do banco local(Falhando)
CTRL + C                    # Para de rodar o projeto
```

## Portas

| Serviço    | URL                   |
| ---------- | --------------------- |
| App (Next) | http://localhost:3000 |
| Postgres   | localhost:5432        |

## Credenciais locais

Só valem no ambiente de desenvolvimento.

| Login                  | Senha          | Perfil |
| ---------------------- | -------------- | ------ |
| sarto@gmail.com        | sarto@123456   | User   |
| domingos@teste.com     | Domingos2026   | User   |
| tomazholanda@teste.com | tomaz@123456   | User   |
| eunicio@teste.com      | eunicio@123456 | User   |
| guilherme@teste.com    | Guilherme2026  | User   |
| dfluisa@mkt-action.com | dfluisa@2026   | User   |
https://2026c01.mkt-action.com/

## Branches

`main` (produção) ← PR ← `dev` (integração) ← PR ← branch do dev (ex.: `lucasdev`)
