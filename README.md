# Branch Protection

Proteção de push direto na main

No GitHub, a forma mais comum é criar uma **Branch protection rule** para a `main`.

## 🔒 Pelo GitHub

1. Entre no seu repositório no  GitHub .
2. Vá em **Settings** → **Branches**.
3. Em **Branch protection rules**, clique em **Add branch ruleset** ou **Add classic branch protection rule** (a interface pode variar).
4. Defina a branch como:
   ```
   main
   ```
5. Recomendo habilitar:
   - ✅ **Require a pull request before merging**
   - ✅ **Require approvals** → pelo menos `1`
   - ✅ **Dismiss stale pull request approvals when new commits are pushed**
   - ✅ **Require status checks to pass before merging**
   - ✅ **Require branches to be up to date before merging**
   - ✅ **Block force pushes**
   - ✅ **Restrict deletions**
6. Salve a regra.

 ### 🛡️ Uma configuração boa para equipe

 O fluxo ficaria:

```
feature/minha-feature
        ↓
      Pull Request
        ↓
   CI / testes passam
        ↓
     Code Review
        ↓
      main
```

 Assim, ninguém consegue simplesmente fazer:

```
git push origin main
```

 e colocar código diretamente na `main`.

 **Dica:** se você usa GitHub Actions, vale configurar os **status checks** para que testes/lint/build sejam obrigatórios antes do merge.

## **Configuração exata recomendada para um projeto com GitHub Actions + PR + 1 aprovação**, passo a passo usando:

- `main` protegida
- Pull Request obrigatório
- **1 aprovação obrigatória**
- GitHub Actions rodando testes
- CI obrigatório para fazer merge
- bloqueio de `force push`
- bloqueio de exclusão da `main`
- branch atualizada antes do merge

 O GitHub atualmente recomenda **Rulesets** para esse tipo de política; eles permitem combinar revisão de PR, status checks e outras proteções.  GitHub Docs+1

 ## 1\. Crie o repositório

 Crie normalmente seu repositório no GitHub.

 Por exemplo:

```
meu-projeto
└── main
```

 Depois clone:

```
git clone git@github.com:SEU_USUARIO/meu-projeto.git
cd meu-projeto
```

---

 ## 2\. Crie o GitHub Actions

 Dentro do projeto, crie:

```
.github/
└── workflows/
    └── ci.yml
```

 Os workflows do GitHub Actions ficam dentro de `.github/workflows` e são definidos em YAML.  GitHub Docs

 Por exemplo, para um projeto Node.js:

```
name: CI

on:
  pull_request:
    branches:
      - main

jobs:
  test:
    name: Tests
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v6

      - name: Setup Node
        uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

 O ponto importante aqui é:

```
on:
  pull_request:
    branches:
      - main
```

 Ou seja:

 > Sempre que alguém abrir/atualizar um PR destinado à `main`, o CI será executado.

 O resultado desse workflow aparecerá no Pull Request como um **status check**. Status checks podem ser usados para impedir o merge enquanto testes ou builds não passarem.  GitHub Docs

---

 ## 3\. Faça o primeiro push

```
git add .
git commit -m "ci: add github actions"
git push origin main
```

 Nesse momento você ainda não precisa ter a `main` protegida.

 Depois disso, vá na aba:

 **Actions**

 e verifique se o workflow executou corretamente.

 Você quer chegar a algo como:

```
CI
└── Tests       ✓
```

---

 # 4\. Agora proteja a `main`

 No GitHub:

 **Repository → Settings → Rules → Rulesets**

 Depois:

 **New ruleset → New branch ruleset**

 A documentação atual do GitHub permite direcionar o ruleset para branches específicas usando padrões como `main`.  GitHub Docs

 Configure:

```
Ruleset name:
Protect main
```

 Em **Target branches**, selecione:

```
Include default branch
```

 ou configure explicitamente:

```
main
```

---

 # 5\. Configure o Pull Request

 Ative:

```
☑ Require a pull request before merging
```

 Depois configure:

```
Required approvals: 1
```

 Assim, ninguém poderá simplesmente fazer:

```
git push origin main
```

 A alteração deverá seguir:

```
feature
   ↓
Pull Request
   ↓
CI
   ↓
1 aprovação
   ↓
main
```

 O GitHub permite exigir uma quantidade específica de aprovações antes do merge.  GitHub Docs+1

---

 # 6\. Configure o CI como obrigatório

 Essa é uma das partes mais importantes.

 Ative:

```
☑ Require status checks to pass
```

 Depois selecione o check:

```
Tests
```

 Esse nome vem daqui:

```
jobs:
  test:
    name: Tests
```

 Então o GitHub deverá mostrar algo parecido com:

```
Required checks

☑ Tests
```

 Agora o comportamento será:

```
PR aberto
   ↓
GitHub Actions
   ↓
Tests
   ↓
    ├── ❌ falhou → NÃO pode fazer merge
    │
    └── ✅ passou
            ↓
       1 aprovação
            ↓
       pode fazer merge
```

 Um status check obrigatório precisa estar em estado bem-sucedido para permitir o merge.  GitHub Docs+1

---

 # 7\. Exija que a branch esteja atualizada

 Eu recomendo também:

```
☑ Require branches to be up to date before merging
```

 Isso evita uma situação como:

```
main
 A---B---C

feature
     \---D
```

 Enquanto isso, outra pessoa colocou:

```
main
 A---B---C---E
```

 Sua feature foi testada contra:

```
A---B---C
```

 mas será integrada em:

```
A---B---C---E
```

 Exigir que a branch esteja atualizada reduz esse risco.

 O GitHub também documenta que, quando essa opção é utilizada junto aos status checks, a branch precisa estar atualizada para que a proteção tenha efeito.  GitHub Docs

---

 # 8\. Bloqueie Force Push

 Ative:

```
☑ Block force pushes
```

 Isso é importante para proteger o histórico da `main`.

 Você não quer que alguém consiga fazer:

```
git push --force origin main
```

 e reescrever o histórico.

---

 # 9\. Bloqueie exclusão da branch

 Ative:

```
☑ Restrict deletions
```

 ou a opção equivalente de impedir exclusão da branch.

 A ideia é simples:

```
main
 ↓
não pode ser deletada
não pode receber push direto
não pode sofrer force push
```

---

 # 10\. Uma configuração que eu recomendo

 Para um projeto pequeno/médio, eu começaria assim:

```
RULESET: Protect main

Target:
  ☑ main

Pull Requests:
  ☑ Require a pull request before merging
  ☑ Required approvals: 1

Status checks:
  ☑ Require status checks to pass
      ☑ Tests
  ☑ Require branches to be up to date

History:
  ☑ Block force pushes
  ☑ Restrict deletions
```

 E **não começaria** ativando todas as opções possíveis.

 Por exemplo, assinatura obrigatória de commits, merge queue, code scanning etc. podem ser adicionados depois conforme a necessidade do projeto. O GitHub possui várias outras regras disponíveis.  GitHub Docs

---

 # 11\. Escolha o tipo de merge

 Eu também recomendo configurar:

```
Allow squash merging:       ✅
Allow merge commits:        ❌
Allow rebase merging:       ❌
```

 Assim você trabalha com:

```
feature/login
      ↓
PR #15
      ↓
Squash and merge
      ↓
main
```

 E a `main` fica com um histórico mais limpo:

```
A
B
C
D - Implement login
E - Implement payment
F - Fix authentication
```

 Em vez de:

```
A
B
Merge branch...
C
Merge pull request...
D
Merge branch...
E
```

 O GitHub oferece `merge commit`, `squash and merge` e `rebase and merge`; a escolha depende da política de histórico desejada.  GitHub Docs

---

 # 12\. Como ficará o fluxo de trabalho

 Depois disso, **ninguém deveria trabalhar diretamente na `main`**.

 O fluxo será:

```
              ┌──────────────┐
              │     main     │
              └──────┬───────┘
                     │
                     │
              criar branch
                     │
                     ▼
            ┌─────────────────┐
            │ feature/login   │
            └────────┬────────┘
                     │
                desenvolver
                     │
                     ▼
                  git push
                     │
                     ▼
            ┌─────────────────┐
            │ Pull Request    │
            │     → main      │
            └────────┬────────┘
                     │
              ┌──────┴───────┐
              │              │
             CI             Review
              │              │
           Tests ✓        1 aprovação
              │              │
              └──────┬───────┘
                     │
                     ▼
               Squash & Merge
                     │
                     ▼
              ┌──────────────┐
              │     main     │
              └──────────────┘
```

 Esse é um **excelente fluxo inicial para um projeto profissional**.

 ### E tem uma coisa especialmente interessante para o seu caso

 Como você estava falando anteriormente sobre **Qualidade de Software → Desenvolvimento**, esse modelo é justamente uma boa forma de transformar conceitos de QA em prática de desenvolvimento:

```
Código
  ↓
Pull Request
  ↓
Automated Tests
  ↓
Code Review
  ↓
Quality Gate
  ↓
main
```

 Você está basicamente transformando **qualidade em uma regra automatizada do processo de desenvolvimento**, em vez de depender apenas de alguém lembrar de executar os testes.

 Documentação oficial: criação de Rulesets no GitHub \
  Documentação oficial: regras disponíveis nos Rulesets
