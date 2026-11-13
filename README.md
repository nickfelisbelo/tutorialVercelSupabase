# Supabase + Vercel 

## Parte 1: Supabase

### 1. Criar o projeto no Supabase

1. Acesse https://supabase.com/ e entre na sua conta.
2. Clique em **New project**.
3. Escolha a organização, o nome do projeto, uma senha forte para o banco e a região.
4. Clique em **Create new project** e aguarde o status ficar **Healthy**.
5. No projeto, acesse **Connect** para visualizar as connection strings.

### 2. Baixar o certificado SSL do Supabase

Esta é a etapa adicional necessária para validar o certificado do servidor PostgreSQL.

1. Abra seu projeto no Supabase.
2. Acesse **Database → Settings** (em algumas versões da interface, a seção pode aparecer como Database Settings).
3. Procure a seção **SSL Configuration**.
4. Localize a opção **Download Certificate**.
5. Baixe o certificado CA disponibilizado pelo Supabase.

Crie a seguinte estrutura na raiz do backend:

```text
back/
├── certs/
│   └── supabase-prod-ca.crt
├── src/
├── prisma/
├── server.js
├── package.json
└── .env
```

Coloque o certificado baixado dentro de `certs/` e renomeie-o para:

`supabase-prod-ca.crt`

**Atenção:** não altere o conteúdo do certificado. Ele deve continuar contendo o cabeçalho e o rodapé PEM, normalmente:

```text
-----BEGIN CERTIFICATE-----
CONTEUDO DO CERTIFICADO
-----END CERTIFICATE-----
```

O certificado CA é público, mas deve ser obtido diretamente do painel do seu próprio projeto Supabase. Não utilize certificados encontrados em repositórios aleatórios.

### 3. Obter as URLs corretas do PostgreSQL

Para utilizar o Prisma com a Vercel, normalmente serão necessárias duas conexões:

- **Transaction Pooler — porta 6543:** conexão da aplicação em ambiente serverless.
- **Session Pooler — porta 5432:** conexão utilizada pelo Prisma CLI para migrations, quando a conexão direta não estiver disponível.

No painel do Supabase:

1. Clique em **Connect**.
2. Selecione **Transaction Pooler** e copie a connection string.
3. Selecione **Session Pooler** e copie a connection string.
4. Preserve os hosts, usuários e parâmetros fornecidos pelo painel.

Crie o arquivo `.env` na raiz do backend:

```env
# Aplicação: Transaction Pooler
DATABASE_URL="postgresql://USUARIO:SENHA@HOST-POOLER:6543/postgres"

# Prisma CLI: Session Pooler
DIRECT_URL="postgresql://USUARIO:SENHA@HOST-POOLER:5432/postgres"
```

**Importante sobre SSL:** neste tutorial, a validação do certificado da aplicação será configurada diretamente no driver `pg`, utilizando o arquivo CA. Por isso, não adicione `sslmode`, `sslrootcert` ou outros parâmetros SSL à `DATABASE_URL`: parâmetros SSL presentes na connection string podem substituir as opções SSL fornecidas ao driver.

Para as migrations, configure a validação do certificado na connection string `DIRECT_URL` conforme a seção 6.

#### Senhas com caracteres especiais

Caracteres reservados em uma URL, como `@`, `#`, `%`, `/` e `?`, precisam ser codificados com percent-encoding na parte da senha.

Por exemplo, `@` deve ser convertido em `%40`.

A alternativa mais simples é gerar uma senha alfanumérica forte.

#### Proteger o arquivo `.env`

Adicione ao `.gitignore`:

```gitignore
.env
.env.*
!.env.example
```

Se necessário, crie um `.env.example` com valores fictícios.

### 4. Instalar as dependências compatíveis com Prisma 7

Na raiz do backend, execute:

```bash
npm install @prisma/client@^7 @prisma/adapter-pg@^7 pg dotenv
npm install -D prisma@^7
```

Mantenha `prisma`, `@prisma/client` e `@prisma/adapter-pg` na mesma versão principal.

Para conferir:

```bash
npm ls prisma @prisma/client @prisma/adapter-pg pg
```

### 5. Configurar o Prisma 7

#### 5.1. Arquivo `prisma.config.ts`

Este arquivo deve ficar na raiz do backend, ao lado do `package.json`.

```ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: env("DIRECT_URL"),
  },
});
```

O Prisma CLI utilizará `DIRECT_URL` para executar as migrations.

#### 5.2. Arquivo `prisma/schema.prisma`

No Prisma 7, a URL fica no `prisma.config.ts`, não no bloco `datasource`.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}
```

Mantenha abaixo os seus próprios `model` e `enum`. Não substitua seus modelos pelos exemplos deste tutorial.

#### 5.3. Cliente Prisma com certificado SSL

Agora configure o arquivo `src/data/prisma.js`.

Esta é a parte mais importante para resolver os erros de validação do certificado no Prisma 7.

```js
require("dotenv/config");

const fs = require("node:fs");
const path = require("node:path");

const { PrismaClient } = require("@prisma/client");
const { PrismaPg } = require("@prisma/adapter-pg");

const certificado = fs.readFileSync(
  path.resolve(__dirname, "../../certs/supabase-prod-ca.crt"),
  "utf8"
);

const adapter = new PrismaPg({
  connectionString: process.env.DATABASE_URL,
  ssl: {
    ca: certificado,
    rejectUnauthorized: true
  }
});

const prisma = new PrismaClient({ adapter });

module.exports = prisma;
```
### 6. Criar e aplicar as migrations

Primeiro valide o schema:

```bash
npx prisma validate
```

Para criar e aplicar uma migration no banco de desenvolvimento:

```bash
npx prisma migrate dev --name init
```

Execute:

```bash
npx prisma generate
```
Aplique as migrations pendentes a partir de um ambiente controlado, usando a URL de sessão/direta configurada para o CLI:

```bash
npx prisma migrate deploy
```


### Parte 2: Subir o back na Vercel
Antes de tudo crie na raiz da pasta um arquivo chamado ``vercel.json`` e uma pasta chamada ``api`` com um arquivo chamado ``index.js`` dentro dele.<br><br>
**vercel.json:**
```vercel.json
{
  "version": 2,
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/api/index"
    }
  ]
}
```
**api/index.js:**
```index.js
const app = require("../server");

module.exports = app;
```
Agora, siga os passos:
1. Entre na [Vercel](https://vercel.com/)<br>
2. Selecione **Create new**<br>
   ![Imagem 1 Vercel](./assets/vercel1.png)<br>
3. Selecione **Project**<br>
   ![Imagem 2 Vercel](./assets/vercel2.png)<br>
4. Selecione o repositório com o Backend que deseja hospedar online<br>
   ![Imagem 3 Vercel](./assets/vercel3.png)<br>
5. Selecione o **Other** em Application Preset <br>
   ![Imagem 4 Vercel](./assets/vercel4.png)<br>
6. Em **Build and Output Settings** habilite *Install Command* e escreva o comando:
   ```cmd
   npm install
   ```
   ![Imagem 5 Vercel](./assets/vercel5.png)
7. Importe o ``.env`` em Environment Variables <br>
   ![Imagem 6 Vercel](./assets/vercel6.png)
8. Selecione **Create Project**
9. Selecione Deploy<br>
    ![Imagem 7 Vercel](./assets/vercel7.png)
10. Veja se deu certo 
    ![Imagem 8 Vercel](./assets/vercel8.png)