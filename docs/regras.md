# Regras do Projeto Exped

Convenções e regras de colaboração entre o time e o Claude Code neste repositório.

---

## Idioma

- Todo o conteúdo do projeto — código, comentários, documentação, commits, PRs e conversas — deve ser em **português**.
- Exceções: nomes de bibliotecas, termos técnicos sem tradução consagrada (e.g. `deploy`, `branch`, `commit`) e identificadores de código (variáveis, funções, classes) podem permanecer em inglês quando for a convenção da tecnologia adotada.

---

## Colaboração com o Claude

- O Claude deve sempre responder em português.
- Antes de escrever código, o projeto deve passar pelas fases de discovery, requisitos e arquitetura (ver [skills.md](skills.md)).
- O Claude não deve propor implementações de código enquanto o produto não tiver sido suficientemente descoberto e documentado.
- Decisões importantes de produto e arquitetura devem ser registradas em documentos dentro de `docs/`.

---

## CLAUDE.md

- O `CLAUDE.md` deve conter **apenas índices**: uma linha por seção com o link para o documento correspondente em `docs/`.
- Descrições, listas detalhadas e conteúdo explicativo pertencem aos arquivos em `docs/`, nunca diretamente no `CLAUDE.md`.
- Sempre que um novo tema relevante surgir, crie um arquivo em `docs/` e adicione apenas o link no `CLAUDE.md`.

---

## Skills

- Antes de iniciar qualquer tópico novo — produto, requisitos, arquitetura, UX, domínio — **sempre busque uma skill relevante** com `/find-skills <tópico>`.
- Se uma skill adequada for encontrada, instale-a globalmente com `npx skills add <pacote> -g -y` e registre-a em `docs/skills.md` antes de usá-la.
- **Sempre que uma nova skill for instalada ou descoberta, adicione imediatamente uma entrada em `docs/skills.md`** com: nome, pacote, como invocar e quando usar. Sem exceções — skill instalada sem registro em `docs/skills.md` está incompleta.
- Nunca resolva um tópico especializado sem verificar se existe uma skill disponível que cubra o assunto.

---

## Estrutura de documentos

- `docs/regras.md` — este arquivo; regras e convenções do projeto
- `docs/skills.md` — skills instaladas e ordem de uso recomendada
- `docs/brainstorming.md` — índice do material de ideação e referência do produto
- `docs/` — demais documentos de produto, requisitos e arquitetura gerados ao longo do projeto

---

## Fluxo de alterações

- Ao concluir qualquer alteração em arquivos do projeto, o Claude deve **perguntar se o usuário deseja fazer commit e push** antes de encerrar a resposta.
- **NUNCA commitar ou fazer push automaticamente sem a confirmação explícita do usuário.** Esta regra não tem exceções — nem quando a alteração parece óbvia, nem ao final de uma sessão de trabalho, nem após criar ou editar documentos.
- Esta regra se aplica a qualquer operação git que modifique o histórico: `git commit`, `git push`, `git merge`, `git rebase`, `git tag`.

---

## Git

Este projeto adota o padrão **[Conventional Commits](https://www.conventionalcommits.org/)** para todas as mensagens de commit.

### Formato

```
<tipo>[escopo opcional]: <descrição>

[corpo opcional]

[rodapé(s) opcional(is)]
```

### Tipos permitidos

| Tipo       | Uso                                                                 |
|------------|---------------------------------------------------------------------|
| `feat`     | Nova funcionalidade                                                 |
| `fix`      | Correção de bug                                                     |
| `docs`     | Alterações apenas em documentação                                   |
| `style`    | Formatação, ponto e vírgula faltando etc. (sem mudança de lógica)  |
| `refactor` | Refatoração de código sem adição de feature ou correção de bug     |
| `test`     | Adição ou correção de testes                                        |
| `chore`    | Atualização de tarefas de build, configurações etc.                 |
| `perf`     | Melhoria de performance                                             |
| `ci`       | Mudanças em arquivos e scripts de CI/CD                             |
| `revert`   | Reverter um commit anterior                                         |

### Regras

- A descrição deve ser em **português**, no imperativo e sem letra maiúscula no início (e.g. `feat: adiciona modelo de domínio de entregas`).
- O escopo é opcional e deve identificar o módulo ou contexto afetado (e.g. `feat(entregas): adiciona cálculo de rota`).
- Quebras de compatibilidade (_breaking changes_) devem ser indicadas com `!` após o tipo/escopo (e.g. `feat!: altera estrutura da API`) e detalhadas no rodapé com `BREAKING CHANGE:`.
- Branches nomeadas em português com kebab-case (e.g. `feat/discovery-modelo-de-negocio`).
- PRs devem referenciar o documento de requisitos ou decisão de arquitetura correspondente quando houver.

### Skill instalada

A skill `patricio0312rev/skills@conventional-commits` está instalada globalmente para auxiliar na criação de commits seguindo este padrão.
