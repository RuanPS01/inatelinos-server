# inatelinos-server

Repositório de **configuração do Firebase** do Inatelinos: regras de segurança
do Firestore e do Storage, e índices do Firestore. Não há código de aplicação
aqui — é a **fonte da verdade** da configuração de backend, com deploy
automático via GitHub Actions.

> App mobile: [`inatelinos-app`](https://github.com/RuanPS01/inatelinos-app) ·
> Web: [`inatelinos-web`](https://github.com/RuanPS01/inatelinos-web)

## Conteúdo

| Arquivo | O que é |
|---|---|
| `firestore.rules` | Regras de segurança do Firestore |
| `storage.rules` | Regras de segurança do Cloud Storage |
| `firestore.indexes.json` | Índices do Firestore (inclui os de collection group) |
| `firebase.json` | Aponta o Firebase CLI para os arquivos acima |
| `.firebaserc` | Projeto padrão do Firebase |

Todas as regras exigem, no servidor: usuário autenticado, e-mail do domínio
Inatel (`@inatel.br` ou `@sigla.inatel.br`) **e** e-mail confirmado
(`email_verified == true`).

## Pipeline de deploy 🚀

O workflow [`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)
publica as regras e os índices automaticamente **ao dar merge/push na branch
`main`** (apenas quando um arquivo de configuração muda). Também pode ser
disparado manualmente pela aba **Actions → Deploy Firebase config → Run
workflow**.

Fluxo recomendado: crie uma branch, altere as regras/índices, abra um PR e,
ao aprovar e mergear na `main`, o deploy acontece sozinho.

O workflow tem dois passos:
1. **Firestore (regras + índices)** — funciona no plano gratuito (Spark).
2. **Storage (regras)** — passo **opcional** que não derruba o pipeline. Só
   é publicado depois que o Storage estiver ativado no projeto (veja abaixo).

### Ativar o Cloud Storage (necessário para o deploy do Storage)

O deploy do Storage falha com *"Firebase Storage has not been set up on
project"* enquanto o bucket não for inicializado **uma vez** no console:

- Firebase Console → **Build → Storage → Get Started**.

Sobre o plano: o Firestore roda tranquilo no **Spark (grátis)**. Já a ativação
do **Cloud Storage** passou a exigir o plano **Blaze** em projetos novos — o
console avisa na hora do *Get Started*. O Blaze tem **franquia gratuita**
(Storage: ~5 GB armazenados e cotas diárias de download/upload) e só cobra
acima disso, mas exige vincular uma **conta de faturamento** (cartão). O app
e a web usam o Storage para **upload de imagens** (posts, stories e fotos de
perfil); sem ele, todo o resto funciona, mas os envios de imagem ficam
indisponíveis até o Storage ser ativado. Ativado o Storage, rode o workflow
novamente (**Actions → Deploy Firebase config → Run workflow**) para publicar
as regras.

### Configuração única dos secrets (obrigatório)

O deploy usa uma **conta de serviço** do Google Cloud. Configure dois secrets
no repositório em **Settings → Secrets and variables → Actions → New
repository secret**:

1. **`FIREBASE_PROJECT_ID`** — o ID do seu projeto no Firebase
   (ex.: `inatelinos`).

2. **`FIREBASE_SERVICE_ACCOUNT`** — o JSON completo de uma conta de serviço
   com permissão de deploy. Para obtê-lo:
   - [console.cloud.google.com](https://console.cloud.google.com) → selecione
     o projeto → **IAM & Admin → Service Accounts → Create service account**.
   - Conceda os papéis:
     - **Firebase Rules Admin** (`roles/firebaserules.admin`)
     - **Cloud Datastore Index Admin** (`roles/datastore.indexAdmin`)
     - **Firebase Admin** (mais simples, cobre tudo) — ou os dois acima para
       um acesso mínimo.
   - Em **Keys → Add key → Create new key → JSON**, baixe o arquivo e cole
     todo o conteúdo no valor do secret.

> ⚠️ Nunca faça commit do JSON da conta de serviço. O `.gitignore` já bloqueia
> arquivos `*service-account*.json`.

### Editar o `.firebaserc`

Troque `SEU_PROJECT_ID_DO_FIREBASE` pelo ID real do projeto (opcional — o
workflow já passa `--project` a partir do secret `FIREBASE_PROJECT_ID`, mas
manter o `.firebaserc` correto facilita rodar o Firebase CLI localmente).

## Uso local (opcional)

```bash
npm install -g firebase-tools
firebase login

# Testar as regras / ver o que mudaria
firebase deploy --only firestore:rules,firestore:indexes,storage --project SEU_PROJECT_ID

# Emuladores para desenvolvimento
firebase emulators:start --only firestore,storage
```

## Como adicionar um índice composto

Ao criar uma query que o Firestore exija índice, ele retorna no erro um link
que gera o índice — ou adicione manualmente em `firestore.indexes.json`, no
array `indexes`. Exemplo:

```json
{
  "indexes": [
    {
      "collectionGroup": "posts",
      "queryScope": "COLLECTION_GROUP",
      "fields": [
        { "fieldPath": "owner_email", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    }
  ]
}
```

Faça o merge na `main` e o índice é criado no deploy.
