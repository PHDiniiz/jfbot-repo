# Contribuindo

Obrigado por contribuir com este projeto.

## Código de conduta

- Respeite todas as pessoas envolvidas no projeto.
- Seja objetivo em discussões técnicas.
- Não use linguagem ofensiva, discriminatória ou hostil.

## Como abrir issues

Use os templates em `.github/ISSUE_TEMPLATE/`:

- `bug_report.md` para falhas e comportamentos inesperados
- `feature_request.md` para melhorias e novas funcionalidades

Inclua sempre:

- contexto
- impacto
- passos claros para reproduzir (quando aplicável)
- ambiente de execução

## Fluxo de Pull Request

1. Crie uma branch a partir de `main`:
   - exemplo: `feat/nome-curto` ou `fix/nome-curto`
2. Faça alterações pequenas e coesas.
3. Atualize documentação e testes quando necessário.
4. Abra PR usando `.github/PULL_REQUEST_TEMPLATE.md`.
5. Aguarde revisão e ajuste os pontos solicitados.
6. Merge somente após aprovação.

## Política de proteção da branch `main`

A branch `main` deve permanecer protegida.

Resumo:

| Regra | Valor |
| --- | --- |
| Push direto em `main` | Bloqueado |
| Aprovação obrigatória | 1 review |
| Force push | Desabilitado |
| Delete branch protegida | Desabilitado |

## Configuração inicial (uma vez)

O workflow `.github/workflows/branch-protection.yml` aplica proteção na `main` via API GitHub.

Pré-requisito:

- criar secret de repositório `ADMIN_TOKEN` com:
  - PAT classic com escopo `repo`, ou
  - PAT fine-grained com permissão `Administration: Read and write`

Depois:

1. Acesse `Actions` no GitHub.
2. Execute manualmente o workflow `Branch Protection`.
