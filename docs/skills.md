# Skills instaladas — Projeto Exped

Skills do ecossistema [skills.sh](https://skills.sh/) instaladas globalmente para apoiar o planejamento e desenvolvimento da plataforma de logística Exped.

---

## Brainstorming

### `brainstorming`
**Pacote:** `obra/superpowers@brainstorming` · 435.7K installs  
**Como invocar:** `/brainstorming`

Skill de referência do ecossistema para sessões de ideação estruturada. Facilita a exploração livre de ideias sobre features, modelo de negócio, posicionamento e diferenciais competitivos antes de convergir para uma direção. Use no início de qualquer novo épico ou decisão de produto.

---

### `product-brainstorming`
**Pacote:** `anthropics/knowledge-work-plugins@product-brainstorming` · 6.1K installs · *Anthropic oficial*  
**Como invocar:** `/product-brainstorming`

Focado especificamente em brainstorming de produto: explora espaços de problema, gera hipóteses, desafia premissas e funciona como sparring partner para o PM antes de convergir em uma direção. Complementa o `brainstorming` com uma lente orientada a produto.

---

## Product Discovery

### `jobs-to-be-done`
**Pacote:** `wondelai/skills@jobs-to-be-done` · 7.6K installs  
**Como invocar:** `/jobs-to-be-done`

Aplica o framework Jobs-to-be-Done (JTBD) para descobrir o que os clientes realmente precisam — o "trabalho" que contratam um produto para fazer. Essencial antes de definir features: cobre entrevistas JTBD, análise de competição e roadmaps orientados a jobs reais dos usuários. Use para entender por que remetentes, destinatários e transportadoras escolheriam (ou abandonariam) a Exped.

---

## Product Management & PRD

### `prd`
**Pacote:** `github/awesome-copilot@prd` · 24.7K installs · *GitHub oficial*  
**Como invocar:** `/prd`

Gera Product Requirements Documents (PRDs) estruturados e de alta qualidade, incluindo resumo executivo, histórias de usuário, especificações técnicas e análise de riscos. Referência de mercado para documentação de produto. Use para formalizar cada iniciativa da Exped antes de entrar em desenvolvimento.

---

## Análise de Requisitos

### `requirements-analysis`
**Pacote:** `jwynia/agent-skills@requirements-analysis` · 2.3K installs  
**Como invocar:** `/requirements-analysis`

Diagnostica problemas de requisitos e guia a descoberta das reais necessidades e restrições do sistema. Identifica stakeholders, casos de uso, requisitos funcionais e não-funcionais, e ambiguidades que precisam ser resolvidas. Use após o discovery, antes de escrever o PRD final ou começar a arquitetura.

---

### `software-architecture`
**Pacote:** `sickn33/agentic-awesome-skills@software-architecture` · 3.7K installs  
**Como invocar:** `/software-architecture`

Suporte para decisões de arquitetura de software: diagramas, trade-offs entre estilos arquiteturais (monolito, microserviços, event-driven), ADRs (Architecture Decision Records) e revisão de design técnico. Use após o DDD para concretizar a arquitetura da plataforma.

---

## UX / UI

### `design-mobile-apps`
**Pacote:** `designed-by-ai/skills@design-mobile-apps` · 1.5M installs  
**Como invocar:** `/design-mobile-apps`

Guia completo para design de aplicativos mobile: padrões de navegação, componentes, fluxos de telas, tipografia e acessibilidade. Essencial para o app dos entregadores da Exped.

---

### `ui-ux-designer`
**Pacote:** `sickn33/agentic-awesome-skills@ui-ux-designer` · 3.2K installs  
**Como invocar:** `/ui-ux-designer`

Atua como designer UX/UI especializado: pesquisa de usuários, wireframes, protótipos, sistemas de design e heurísticas de usabilidade. Use para o painel web dos gestores e para revisar fluxos de tela do app.

---

### `journey-map`
**Pacote:** `owl-listener/designer-skills@journey-map` · 2.6K installs  
**Como invocar:** `/journey-map`

Mapeia a experiência completa de uma persona: etapas, pontos de contato, emoções e pontos de dor. Use na Etapa 4 do planejamento para mapear as jornadas dos gestores, entregadores e destinatários.

---

## Frontend — React

### `tailwind-v4-shadcn`
**Pacote:** `secondsky/claude-skills@tailwind-v4-shadcn` · 8.3K installs  
**Como invocar:** `/tailwind-v4-shadcn`

Setup produtivo para Tailwind CSS v4 com shadcn/ui, Vite e React. Cobre o padrão `@theme inline`, arquitetura de CSS variables, dark mode com ThemeProvider, composição de componentes, configuração do `vite.config`, armadilhas comuns do v4 e migração do v3. Use ao inicializar o projeto frontend ou ao resolver problemas de tema e dark mode.

---

### `react-doctor`
**Pacote:** `millionco/react-doctor@react-doctor` · 22.8K installs  
**Como invocar:** `/react-doctor`  
**⚠️ Aviso:** Snyk sinalizou risco crítico neste pacote — use com cautela e revise o que a skill sugere antes de aplicar.

Diagnóstico de performance em aplicações React: detecta re-renders desnecessários, sugere uso de `useMemo`, `useCallback` e `memo`, e orienta sobre otimizações de renderização. Use ao perceber lentidão no painel do lojista ou no PWA do entregador.

---

## Git & Convenções

### `conventional-commits`
**Pacote:** `patricio0312rev/skills@conventional-commits` · 505 installs  
**Como invocar:** `/conventional-commits`

Guia para criação de mensagens de commit no padrão Conventional Commits: tipos (`feat`, `fix`, `docs`, `chore`…), escopo, breaking changes e changelog automático. Use ao criar qualquer commit no projeto.

---

## Arquitetura

### `domain-driven-design`
**Pacote:** `wondelai/skills@domain-driven-design` · 7.8K installs  
**Como invocar:** `/domain-driven-design`

Aplica o framework DDD para modelar o domínio do negócio: bounded contexts, agregados, entidades, value objects, repositórios e domain events. Essencial antes de codificar — garante que a linguagem do código reflita a linguagem do negócio. Use para refinar os agregados da Exped (Expedição, Entrega, Ponto, Comprovante) e definir os bounded contexts.

---

### `github-actions-docs`
**Pacote:** `xixu-me/skills@github-actions-docs` · 13 installs  
**Como invocar:** `/github-actions-docs`

Referência de sintaxe e boas práticas para pipelines GitHub Actions: triggers, jobs, steps, secrets, environments e matrizes. Use ao construir ou revisar a esteira de CI/CD do Exped (lint → test → build → deploy no Droplet).

---

### `clean-architecture`
**Pacote:** `clean-architecture` (instalada globalmente)  
**Como invocar:** `/clean-architecture`

Estrutura o software em torno da Dependency Rule: dependências de código apontam para dentro, do framework → casos de uso → entidades. Cobre ports & adapters (hexagonal), onion architecture, separação de camadas e princípios SOLID. Use ao decidir onde a lógica de negócio vive, como isolar o domínio da infraestrutura e como estruturar os módulos sem acoplar ao framework.

---

## Domínio de Logística

### `last-mile-delivery`
**Pacote:** `kishorkukreja/awesome-supply-chain@last-mile-delivery` · 76 installs  
**Como invocar:** `/last-mile-delivery`

Conhecimento especializado em logística de last-mile: rotas de entrega, janelas de tempo, densidade de pedidos, otimização de frota, falhas de entrega e reentregas. Uso recomendado para validar decisões de produto e arquitetura com a realidade operacional do setor.

---

## Ordem de uso recomendada

Alinhada com as etapas do [planejamento.md](planejamento.md):

```
Etapa 1 — Análise do produto
  1. /brainstorming          → Ideação inicial, espaço do problema
  2. /product-brainstorming  → Refinamento da oportunidade de produto
  3. /jobs-to-be-done        → Descoberta do que os usuários realmente precisam

Etapa 2–3 — Módulos e requisitos
  4. /requirements-analysis  → Formalização de requisitos funcionais e não-funcionais
  5. /prd                    → Documento de requisitos do produto

Etapa 4 — Fluxos e jornadas
  6. /journey-map            → Mapeamento da experiência de cada persona
  7. /ui-ux-designer         → Wireframes, fluxos de telas e usabilidade
  8. /design-mobile-apps     → Design do app mobile dos entregadores

Etapa 5 — Arquitetura
  9. /domain-driven-design   → Modelagem do domínio e bounded contexts
 10. /software-architecture  → Decisões e diagramas de arquitetura técnica
 11. /clean-architecture     → Estrutura em camadas e separação de responsabilidades

Etapa 6–7 — Desenvolvimento e CI/CD
 12. /github-actions-docs    → Esteira de CI/CD no GitHub Actions
 13. /conventional-commits   → Mensagens de commit padronizadas

Frontend
 14. /tailwind-v4-shadcn     → Setup e boas práticas shadcn/ui + Tailwind v4
 15. /react-doctor           → Diagnóstico de performance React (re-renders, memoização)

Validação contínua
 16. /last-mile-delivery     → Validação com conhecimento do domínio logístico
```
