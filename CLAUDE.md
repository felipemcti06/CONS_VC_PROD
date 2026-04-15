# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Contexto do Projeto

- **Ferramenta:** IBM Planning Analytics / TM1
- **Cliente:** Votorantim Cimentos — módulo de Consolidação (CONS_VC)
- **Extensões trabalhadas:** `.ti` (Turbo Integrator), `.rules` (regras de cubo), `.json` (metadados), `.hierarchies`
- **Ambiente:** TM1 Server 12, gerenciado via `tm1project.json`

## Estrutura do Repositório

```
cubes/        → Definições de cubos (.json, .rules, .views)
dimensions/   → Definições de dimensões (.json, .hierarchies)
processes/    → Processos TI (.ti + .json de metadados)
chores/       → Agendamentos de processos (.json)
files/        → Arquivos de entrada/saída usados pelos processos
tm1project.json → Manifesto do projeto TM1 (inclui lista de objetos ignorados)
```

## Arquitetura do Modelo

### Prefixos de Módulo

| Prefixo | Módulo |
|---------|--------|
| `ALL`   | Camada transversal — controles, assumptions, uploads status |
| `DFS`   | Demonstrações Financeiras (P&L, Resultados, Fluxo de Caixa, Capital de Giro, Dívida, KPIs) |
| `REV`   | Receita Consolidada |
| `STG`   | Staging — área de entrada de dados brutos |
| `MAP`   | Mapeamentos entre dimensões (Produtos, Contas, Regiões) |
| `SYS`   | Sistema — controle de cargas, time travel, cópia de versões, backup |
| `COS`   | Custo |
| `zCTI`  | Utilitários e templates reutilizáveis da CTI Global |
| `}`     | Objetos de sistema TM1 (não editar sem aprovação explícita) |

### Fluxo de Dados

1. **Entrada:** Arquivos Excel/CSV carregados via processos `STG.001x.*` para o cubo `STG.100.DataBase_Uploads`
2. **Staging → Cubos de negócio:** Processos `ALL.*`, `DFS.*`, `REV.*`, `COS.*` leem do STG e gravam nos cubos analíticos
3. **Mapeamentos:** Cubos `MAP.*` fazem a correspondência entre dimensões do sistema de origem e dimensões TM1
4. **Dimensões:** Atualizadas por processos `ALL.D.*` a partir de cubos de mapa
5. **Cópia de Versão:** Orquestrada pelos processos `zCTI.Version.Copy.*` / `zCTI.001.*`, com log em `SYS.915.Version_Copy_Log` e validação em `SYS.925.Copy_Validation`

### Cubos Principais

- `STG.100.DataBase_Uploads` — staging de entrada de dados
- `DFS.100.Profit_and_Loss` — P&L consolidado
- `DFS.120.Financial_Results` — Resultados financeiros
- `DFS.140.Indirect_Cash_Flow` — Fluxo de caixa indireto
- `DFS.160.Working_Capital` — Capital de giro
- `DFS.180.Gross_Debt` — Dívida bruta
- `DFS.200.KPIs` — Indicadores
- `REV.100.Revenue_Consolidated` — Receita
- `ALL.001.Actual_Control` — Controle de realizado
- `ALL.002.Uploads_Status` — Status de cargas
- `SYS.200.Time_Travel` — Snapshots de versão

## Padrões de Nomenclatura

- **Processos de negócio:** `MODULO.NNNN.NN.Descricao_CuboAlvo` (ex: `STG.0010.10.Carga_Realizado_STG.100.DataBase_Uploads`)
- **Processos de atualização de dimensão:** `ALL.D.NNNN.NN.Atualiza_NomeDimensao`
- **Processos utilitários CTI:** `zCTI.Descricao` ou `zCTI.Modulo.NNN.Descricao`
- **Variáveis locais TI:** `vNomeVariavel`
- **Parâmetros de processo:** `pNomeParametro`
- **Subsets temporários:** `tmp_NomeSubset`
- **Views temporárias:** `tmp_NomeView`
- **Objetos de sistema TM1:** prefixo `}` — nunca modificar sem aprovação explícita

## Regras Absolutas

- NUNCA modificar objetos com prefixo `}` sem aprovação explícita
- NUNCA fazer push direto na branch `main` — sempre via PR
- SEMPRE criar branch antes de qualquer alteração
- SEMPRE validar feeders em arquivos `.rules` que referenciam elementos calculados
- Objetos listados em `tm1project.json` sob `"Ignore"` não são versionados — não recriar nem editar esses cubos/dimensões

## Git & GitHub

- Branch atual de desenvolvimento: `develop`
- Branch protegida: `main` — merge somente após aprovação humana
- Padrão de branches: `feature/descricao`, `fix/descricao`, `refactor/descricao`
- Commits em português, descritivos: `"feat: adiciona processo de carga de KPI"`
- PRs com descrição do impacto no modelo TM1

## Comunicação

- Respostas em português
- Comentários no código em português
- Explicações técnicas objetivas e diretas
