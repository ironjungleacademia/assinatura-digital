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

### Se a tabela `documentos` já existir (atualização com verificação de identidade)

Se você já criou a tabela antes, rode este SQL adicional para acrescentar os campos de verificação por CPF/data de nascimento e o registro de IP/navegador na assinatura:

```sql
alter table documentos add column if not exists aluno_cpf text;
alter table documentos add column if not exists aluno_nascimento text;
alter table documentos add column if not exists ip_assinatura text;
alter table documentos add column if not exists user_agent_assinatura text;
alter table documentos add column if not exists titulo text;
alter table documentos add column if not exists assinatura_pagina int;
alter table documentos add column if not exists assinatura_x float8;
alter table documentos add column if not exists assinatura_y float8;
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

- Antes de abrir o documento, o aluno precisa confirmar **CPF e data de nascimento** cadastrados por você no envio — isso vincula a assinatura à identidade real do aluno, não só a quem tiver o link.
- No momento da assinatura, o sistema também grava automaticamente o **IP** e o **navegador/dispositivo** usados, como reforço de prova.
- Ainda assim, **não é uma assinatura digital certificada (ICP-Brasil)** — é uma rubrica desenhada com verificação de identidade básica, com valor probatório similar ao de uma assinatura física acompanhada de conferência de documento. Para termos de cancelamento internos isso costuma ser suficiente, mas se precisar de validade jurídica mais forte (ex: contestação judicial), vale considerar certificado digital via gov.br.

## Próximos passos possíveis

- Notificação automática (WhatsApp/e-mail) quando o aluno assina
- Expiração automática de links não assinados após X dias
- Múltiplos documentos por aluno numa mesma tela
