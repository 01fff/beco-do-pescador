# Beco do Pescador — Guia de Início do Projeto

> Vitrine digital com painel admin e gestão de torneios.  
> Stack: **Next.js + TypeScript + Supabase + Tailwind CSS + shadcn/ui**

---

## 1. Pré-requisitos

Antes de começar, certifique-se de ter instalado:

- Node.js 20+
- npm ou pnpm
- Git
- Conta na [Vercel](https://vercel.com)
- Conta no [Supabase](https://supabase.com)

---

## 2. Criando o projeto

```bash
npx create-next-app@latest beco-do-pescador \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*"

cd beco-do-pescador
```

### Instalar dependências

```bash
# UI
npx shadcn@latest init

# Supabase
npm install @supabase/supabase-js @supabase/ssr

# Formulários e validação
npm install react-hook-form zod @hookform/resolvers
```

---

## 3. Variáveis de ambiente

Crie o arquivo `.env.local` na raiz do projeto:

```env
NEXT_PUBLIC_SUPABASE_URL=https://<seu-projeto>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<sua-anon-key>
# service_role apenas se absolutamente necessário — nunca expor no frontend
SUPABASE_SERVICE_ROLE_KEY=<sua-service-role-key>
```

> **Atenção:** `.env.local` deve estar no `.gitignore`. As variáveis de produção ficam no painel da Vercel, nunca no repositório.

---

## 4. Estrutura de pastas

Organize o `src/` da seguinte forma após a criação:

```
src/
├── app/
│   ├── page.tsx                              ← home pública
│   ├── produtos/[categoria]/[produto]/page.tsx
│   ├── torneios/[slug]/page.tsx
│   └── admin/
│       ├── layout.tsx                        ← verifica sessão no servidor
│       ├── produtos/
│       ├── torneios/
│       └── inscricoes/
├── components/
│   ├── public/                               ← componentes do site público
│   ├── admin/                                ← componentes do painel
│   └── ui/                                   ← shadcn/ui gerado automaticamente
├── services/
│   ├── produtos.service.ts
│   ├── torneios.service.ts
│   └── inscricoes.service.ts
├── lib/
│   ├── supabase/
│   │   ├── client.ts                         ← para Client Components
│   │   └── server.ts                         ← para Server Components e Actions
│   └── validations/                          ← schemas Zod
├── hooks/
│   └── useAuth.ts
├── types/
└── middleware.ts                             ← proteção das rotas /admin/*
```

---

## 5. Configuração do cliente Supabase

### `src/lib/supabase/client.ts`
```ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### `src/lib/supabase/server.ts`
```ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options)
          );
        },
      },
    }
  );
}
```

---

## 6. Middleware de proteção de rotas

### `src/middleware.ts`
```ts
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => request.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const { data: { user } } = await supabase.auth.getUser();

  if (!user && request.nextUrl.pathname.startsWith("/admin")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return supabaseResponse;
}

export const config = {
  matcher: ["/admin/:path*"],
};
```

---

## 7. Banco de dados no Supabase

### Tabelas a criar

| Tabela | Finalidade |
|---|---|
| `categorias` | Organizar produtos por Pesca, Camping, Náutica etc. |
| `produtos` | Nome, slug, descrição, categoria, status e imagem principal. |
| `produto_imagens` | Galeria de imagens extras por produto. |
| `torneios` | Data, local, regulamento, banner e status. |
| `inscricoes_torneio` | Dados dos participantes inscritos. |
| `perfis_admin` | Extensão de `auth.users` com cargo e permissões (FK obrigatória). |
| `configuracoes` | WhatsApp, redes sociais, endereço. Leitura pública, edição restrita. |

### RLS — regra de ouro

> **Habilitar RLS em todas as tabelas imediatamente após criá-las.**

Políticas mínimas:

```sql
-- Tabelas públicas: leitura livre
CREATE POLICY "select_publico" ON produtos FOR SELECT USING (true);
CREATE POLICY "select_publico" ON categorias FOR SELECT USING (true);
CREATE POLICY "select_publico" ON torneios FOR SELECT USING (true);

-- configuracoes: leitura pública, escrita restrita
CREATE POLICY "select_publico" ON configuracoes FOR SELECT USING (true);
CREATE POLICY "update_admin" ON configuracoes FOR UPDATE
  USING (auth.uid() IS NOT NULL);

-- perfis_admin: restrito ao próprio usuário autenticado
CREATE POLICY "select_proprio" ON perfis_admin FOR SELECT
  USING (auth.uid() = user_id);
CREATE POLICY "update_proprio" ON perfis_admin FOR UPDATE
  USING (auth.uid() = user_id);
```

### service_role key

Evitar sempre que possível. Usar somente no servidor (Server Actions ou Route Handlers) e apenas em operações que realmente exijam bypass do RLS. Nunca expor no frontend.

---

## 8. Módulos do sistema

- **Site público:** Home, sobre, produtos, torneios, contato e localização.
- **Vitrine de produtos:** Categorias, páginas individuais e botão de consulta via WhatsApp.
- **Painel admin:** CRUD de produtos e torneios, ativação/desativação, gestão de inscrições.
- **Inscrições em torneios:** Formulário público + lista de inscritos no painel.
- **WhatsApp:** Número lido da tabela `configuracoes`, nunca hard-coded.

---

## 9. Ordem de desenvolvimento recomendada

1. [ ] Configurar projeto Next.js + dependências
2. [ ] Criar projeto no Supabase e configurar variáveis de ambiente
3. [ ] Criar tabelas e habilitar RLS
4. [ ] Implementar `middleware.ts` e `admin/layout.tsx`
5. [ ] Tela de login do admin (`/login`)
6. [ ] Painel admin — CRUD de categorias e produtos
7. [ ] Site público — vitrine de produtos
8. [ ] Painel admin — CRUD de torneios
9. [ ] Site público — página de torneios + formulário de inscrição
10. [ ] Integração WhatsApp (número via `configuracoes`)
11. [ ] Deploy na Vercel + variáveis de ambiente de produção

---

## 10. Referências

- [Next.js App Router](https://nextjs.org/docs/app)
- [Supabase Docs](https://supabase.com/docs)
- [Supabase Auth + Next.js SSR](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Supabase RLS](https://supabase.com/docs/guides/auth/row-level-security)
- [Next.js Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Tailwind CSS](https://tailwindcss.com)
- [shadcn/ui](https://ui.shadcn.com/docs)
- [React Hook Form](https://react-hook-form.com/docs/useform)
