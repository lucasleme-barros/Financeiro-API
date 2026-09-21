# Financeiro API

Servidor do app **Financeiro**: contas de usuário (só o administrador cria), login e os dados de cada pessoa guardados na nuvem. Feito com Node.js, Express e PostgreSQL.

## Como funciona

- **Contas fechadas:** não existe "criar conta" pública. O administrador cria as contas pelo app (tela **Usuários**). Cada conta nasce com uma **senha temporária**, que a pessoa troca no primeiro acesso.
- **Senhas** ficam guardadas com hash `scrypt` e sal próprio. **Sessões** usam um token aleatório de 64 caracteres; só o hash do token fica no banco. A sessão dura 30 dias.
- **Dados:** cada pessoa tem um documento JSON (pessoal + viagens) com número de versão. O app envia as mudanças e, se dois aparelhos mudarem ao mesmo tempo, a API devolve conflito e o app pergunta qual versão manter.
- **Proteções:** só o site permitido pode chamar a API pelo navegador (CORS), limite de tentativas de login (8 por conta e 40 por IP, a cada 15 minutos), tamanho máximo dos dados de 5 MB, respostas iguais para e-mail que existe ou não.

## Variáveis de ambiente

| Variável | Para quê |
| --- | --- |
| `DATABASE_URL` | Endereço do PostgreSQL (no Railway: `${{Postgres.DATABASE_URL}}`). |
| `ORIGENS_PERMITIDAS` | Endereços do site que podem chamar a API, separados por vírgula. Exemplo: `https://financeiro.vercel.app`. Sem barra no fim. |
| `ADMIN_EMAIL` e `ADMIN_SENHA` | Criam o administrador **só na primeira execução**, quando ainda não existe nenhum usuário. A senha precisa ter 8 caracteres ou mais e é trocada no primeiro login. Depois disso, pode apagar `ADMIN_SENHA`. |
| `PORT` | O Railway define sozinho. |
| `PGSSL` | Use `true` só se o banco exigir SSL (por exemplo, ao usar o endereço público do banco). |

## Publicar no Railway

1. Crie um repositório no GitHub (por exemplo `Financeiro-API`) e envie o conteúdo desta pasta (sem `node_modules`).
2. No Railway: **New Project, Deploy from GitHub repo** e escolha o repositório.
3. No mesmo projeto: **New, Database, Add PostgreSQL**.
4. No serviço da API, aba **Variables**, crie `DATABASE_URL` com o valor `${{Postgres.DATABASE_URL}}`, além de `ORIGENS_PERMITIDAS`, `ADMIN_EMAIL` e `ADMIN_SENHA`.
5. Em **Settings, Networking**, toque em **Generate Domain**. Abra `https://SEU-ENDERECO.up.railway.app/saude`: deve responder `{"ok":true}`.
6. No projeto do site (`Financeiro`), coloque esse endereço em `config.js` (`apiUrl`) e publique de novo.

## Rotas

| Rota | Quem | O que faz |
| --- | --- | --- |
| `GET /saude` | qualquer um | Confere se a API e o banco estão no ar. |
| `POST /auth/login` | qualquer um | Entra com e-mail e senha; devolve o token. |
| `POST /auth/logout` | logado | Encerra a sessão. |
| `GET /auth/eu` | logado | Dados da conta. |
| `POST /auth/trocar-senha` | logado | Troca a senha (derruba as outras sessões). |
| `GET /dados` | logado | Documento da pessoa. Com `?versao=N`, responde `mudou: false` se nada mudou. |
| `PUT /dados` | logado | Salva o documento com `versaoBase`; `409` se outro aparelho salvou antes. Com `forcar: true`, sobrescreve. |
| `GET /admin/usuarios` | administrador | Lista as contas. |
| `POST /admin/usuarios` | administrador | Cria uma conta e devolve a senha temporária. |
| `PATCH /admin/usuarios/:id` | administrador | Ativa ou desativa a conta (desativar derruba as sessões). |
| `POST /admin/usuarios/:id/redefinir-senha` | administrador | Gera nova senha temporária. |

## Rodar e testar no computador

Precisa de Node.js 20 ou mais novo e de um PostgreSQL.

```
npm install
DATABASE_URL=postgresql://usuario:senha@localhost:5432/banco ORIGENS_PERMITIDAS=http://localhost:8000 ADMIN_EMAIL=voce@exemplo.com ADMIN_SENHA=uma-senha-longa npm start
```

Testes (apagam as tabelas do banco indicado, use um banco só para teste):

```
TEST_DATABASE_URL=postgresql://usuario:senha@localhost:5432/banco_de_teste npm test
```

## Cuidados

- Os dados ficam no banco sem criptografia extra (o padrão do PostgreSQL). Faça o backup do app (botão **Backup**) de vez em quando.
- Não há recuperação de senha por e-mail: quem esquecer a senha pede ao administrador para redefinir.
- O limite de tentativas de login fica na memória do servidor e zera quando ele reinicia.
- São dados financeiros de pessoas. Combine com quem vai usar e guarde as credenciais do administrador com cuidado.
