# Guia de Contribuição

Obrigado por querer contribuir com o **jfbot-repo**! Este guia explica como reportar problemas, sugerir melhorias e enviar alterações de forma organizada.

---

## Índice

- [Código de Conduta](#código-de-conduta)
- [Como Reportar um Issue](#como-reportar-um-issue)
- [Como Sugerir uma Funcionalidade](#como-sugerir-uma-funcionalidade)
- [Como Enviar uma Pull Request](#como-enviar-uma-pull-request)
- [Regras da Branch `main`](#regras-da-branch-main)

---

## Código de Conduta

Ao participar deste projeto, você concorda em manter um ambiente respeitoso e colaborativo. Seja objetivo, educado e construtivo em todos os comentários e discussões.

---

## Como Reportar um Issue

Se você encontrou um bug ou comportamento inesperado, abra um **Issue** seguindo os passos abaixo:

1. Acesse a aba [**Issues**](../../issues) do repositório.
2. Clique em **New issue**.
3. Escolha o template **Bug Report** e preencha todas as seções:
   - **Descrição do problema**: o que aconteceu e o que era esperado.
   - **Passos para reproduzir**: sequência mínima de ações que reproduz o bug.
   - **Ambiente**: versão do sistema, linguagem, dependências relevantes.
4. Clique em **Submit new issue**.

> ⚠️ Antes de abrir um novo issue, pesquise os [issues existentes](../../issues) para evitar duplicatas.

---

## Como Sugerir uma Funcionalidade

Para propor uma melhoria ou nova funcionalidade:

1. Acesse a aba [**Issues**](../../issues) do repositório.
2. Clique em **New issue**.
3. Escolha o template **Feature Request** e descreva:
   - O problema que a funcionalidade resolve.
   - A solução proposta com o máximo de detalhes possível.
   - Alternativas que você considerou.
4. Clique em **Submit new issue**.

---

## Como Enviar uma Pull Request

**Pushes diretos na branch `main` são bloqueados.** Toda alteração deve passar por revisão de código via Pull Request. Siga as etapas abaixo:

1. **Fork** o repositório ou crie uma branch a partir de `main`:
   ```bash
   git checkout -b feat/minha-funcionalidade
   ```
2. Faça as alterações necessárias e realize commits descritivos:
   ```bash
   git commit -m "feat: adiciona nova funcionalidade X"
   ```
3. Envie sua branch para o repositório remoto:
   ```bash
   git push origin feat/minha-funcionalidade
   ```
4. Abra uma **Pull Request** para a branch `main`:
   - Acesse a aba [**Pull requests**](../../pulls) e clique em **New pull request**.
   - Preencha o template com uma descrição clara das mudanças.
   - Aguarde a revisão e aprovação de pelo menos **1 revisor**.
5. Após aprovação, o merge será realizado pelo mantenedor do projeto.

---

## Regras da Branch `main`

A branch `main` é protegida pelas seguintes regras:

| Regra | Configuração |
|-------|-------------|
| Push direto | ❌ Bloqueado |
| Merge sem revisão | ❌ Bloqueado |
| Pull Request obrigatória | ✅ Sim |
| Mínimo de aprovações | ✅ 1 revisor |
| Verificações de CI obrigatórias | ✅ Sim (quando configuradas) |

Essas regras garantem a qualidade e rastreabilidade de todas as alterações no código.

### Aplicando as regras via workflow

O repositório inclui o workflow `.github/workflows/branch-protection.yml` que pode ser disparado manualmente para aplicar as regras de proteção via API do GitHub. Para utilizá-lo:

1. Crie um **Personal Access Token (PAT)** com escopo `repo` (classic) ou permissão *Administration: Read and write* (fine-grained).
2. Salve o token como secret `ADMIN_TOKEN` em **Settings → Secrets and variables → Actions → New repository secret**.
3. Acesse a aba **Actions**, selecione o workflow **Setup Branch Protection** e clique em **Run workflow**.
