# Iron Jungle — Assinatura Digital de Termos

Sistema para enviar PDFs (termos de cancelamento) para alunos assinarem com rubrica via link, sem depender do PDFFiller.

Mesmo molde do sistema da rádio: **front-end estático no GitHub Pages + Supabase como backend** (banco de dados + armazenamento de arquivos). Não precisa de servidor rodando.

## Como funciona

1. Você abre `admin.html`, sobe o PDF e digita o nome do aluno
2. O sistema gera um link único (`assinar.html?id=...`) e você envia pro aluno (WhatsApp, etc)
3. O aluno abre o link, visualiza o PDF, desenha a rubrica na tela
4. O navegador do próprio aluno "carimba" a rubrica no PDF (usando `pdf-lib`) e salva o PDF assinado no Supabase
5. Você acompanha tudo pelo `admin.html`, com status pendente/assinado e download do PDF final

Tudo roda no navegador — não existe backend próprio, só o Supabase.

## Passo 1 — Criar o projeto Supabase

Se você já tem o projeto Supabase usado na rádio, pode reaproveisar o mesmo (só criar tabela e bucket novos). Senão, crie um projeto gratuito em supabase.com.

## Passo 2 — Criar a tabela

No SQL Editor do Supabase, rode:

```sql
create table documentos (
  id uuid primary key default gen_random_uuid(),
  aluno_nome text not null,
  pdf_original_path text not null,
  pdf_assinado_path text,
  status text not null default 'pendente',
  created_at timestamptz not null default now(),
  signed_at timestamptz
);

alter table documentos enable row level security;

-- Permite que qualquer pessoa com a chave anon leia e crie/atualize documentos.
-- É o mesmo nível de segurança de um link "secreto" (quem tem o link/id acessa).
-- Suficiente para uso interno da academia, mas não é criptografia de nível bancário.
create policy "allow all select" on documentos for select using (true);
create policy "allow all insert" on documentos for insert with check (true);
create policy "allow all update" on documentos for update using (true);
```

## Passo 3 — Criar o bucket de armazenamento

Em **Storage**, crie um bucket chamado `documentos`, marcado como **público** (leitura pública).

Depois, em Storage → Policies, adicione policies permitindo upload/download com a chave anon (igual à tabela acima) — ou simplesmente marque o bucket como público, que já resolve a leitura; para upload, adicione:

```sql
create policy "allow anon upload" on storage.objects
for insert to anon
with check (bucket_id = 'documentos');

create policy "allow anon read" on storage.objects
for select to anon
using (bucket_id = 'documentos');
```

## Passo 4 — Pegar as credenciais

Em **Project Settings → API**, copie:
- `Project URL`
- `anon public key`

Cole essas duas informações no topo de **ambos** os arquivos `admin.html` e `assinar.html`, nas variáveis `SUPABASE_URL` e `SUPABASE_ANON_KEY`.

## Passo 5 — Publicar no GitHub Pages

1. Crie um repositório (ex: `assinatura-digital`) na organização `ironjungleacademia`, igual ao da rádio
2. Suba os arquivos `admin.html`, `assinar.html`
3. Ative o GitHub Pages nas configurações do repositório (branch `main`, pasta raiz)
4. O admin fica em: `https://ironjungleacademia.github.io/assinatura-digital/admin.html`
5. Os links de assinatura dos alunos vão apontar automaticamente para `assinar.html?id=...` no mesmo domínio

⚠️ Depois de publicar, confira que a variável `BASE_URL_ASSINATURA` no `admin.html` bate com a URL real do `assinar.html` publicado.

## Segurança — pontos a saber

- O acesso é por **link único** (UUID), não por senha. Quem tiver o link acessa o documento — como a maioria das ferramentas simples de assinatura por link.
- O PDF assinado grava data/hora da assinatura como prova, mas **não é uma assinatura digital certificada (ICP-Brasil)** — é uma rubrica desenhada, com valor probatório similar ao de uma assinatura física escaneada. Para termos de cancelamento internos isso costuma ser suficiente, mas se precisar de validade jurídica mais forte, vale considerar isso.
- Para reforçar a prova, dá pra adicionar registro de IP e user-agent do aluno no banco (fica fácil de acrescentar depois).

## Próximos passos possíveis

- Notificação automática (WhatsApp/e-mail) quando o aluno assina
- Expiração automática de links não assinados após X dias
- Múltiplos documentos por aluno numa mesma tela
