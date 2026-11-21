# Módulos Funcionais — Exped

Etapa 2 do planejamento. Produzido com base no [discovery.md](discovery.md) e na arquitetura definida em [arquitetura.md](arquitetura.md).

---

## Índice

1. [Visão Geral dos Módulos](#1-visão-geral-dos-módulos)
2. [M1 — Expedição](#2-m1--expedição)
3. [M2 — Roteirização](#3-m2--roteirização)
4. [M3 — Comprovante e Ocorrência](#4-m3--comprovante-e-ocorrência)
5. [M4 — Compradores](#5-m4--compradores)
6. [M5 — Polo e Pontos](#6-m5--polo-e-pontos)
7. [M6 — Identidade e Acesso](#7-m6--identidade-e-acesso)
8. [M7 — Notificação](#8-m7--notificação-futuro)
9. [Mapa de Dependências entre Módulos](#9-mapa-de-dependências-entre-módulos)
10. [Escopo por Módulo — MVP vs. Futuro](#10-escopo-por-módulo--mvp-vs-futuro)

---

## 1. Visão Geral dos Módulos

| Módulo | Responsabilidade central | MVP |
|---|---|---|
| **M1 — Expedição** | Ciclo de vida das expedições e entregas | ✅ |
| **M2 — Roteirização** | Geração e gestão do plano de entrega | ✅ |
| **M3 — Comprovante e Ocorrência** | Registro de provas de entrega e falhas | ✅ |
| **M4 — Compradores** | Cadastro, busca e histórico de compradores | ✅ |
| **M5 — Polo e Pontos** | Mapa de setores, vagas e excursões | ✅ |
| **M6 — Identidade e Acesso** | Autenticação de lojistas e entregadores | ✅ |
| **M7 — Notificação** | Envio de mensagens ao comprador via WhatsApp/SMS | ⏳ Futuro |

---

## 2. M1 — Expedição

### O que faz

Gerencia o ciclo de vida completo de uma expedição: desde a criação pelo lojista até a conclusão ou devolução de cada entrega individual.

### Campos de uma entrega

| Campo | Tipo | Obrigatório | Observação |
|---|---|---|---|
| **Cliente** | autocomplete | Sim | Busca progressiva nos compradores da loja; opção de cadastrar novo inline se não encontrar |
| **Cidade / Estado** | autocomplete | Sim | Base de cidades externa (IBGE); aceita só cidades válidas |
| **Excursão / Transportadora** | autocomplete + `+` | Sim | Lista de M5; botão `+` abre cadastro rápido sem sair da tela |
| **Volumes** | número | Sim | Padrão 1 |
| **Tamanho** | toggle P / M / G | Sim | Informativo — sem impacto em regras de negócio ou roteirização |
| **Horário limite** | data + hora | Não | Prazo da entrega; usado no cálculo de urgência em M2 |
| **Observação** | texto livre | Não | Campo livre para o lojista |

> Os três campos com autocomplete (Cliente, Cidade, Excursão) exibem sugestões abaixo conforme o lojista digita — a mesma lista de dados que o modo de voz (futuro) usará para resolver termos falados em registros estruturados.

### O que pertence a este módulo

- Abertura e fechamento manual da expedição pelo lojista
- Criação de expedições com uma ou mais entregas
- Associação de comprador, volumes, tamanho, cidade e excursão a cada entrega
- Definição ou atualização do ponto de destino de uma entrega
- Controle de status de cada entrega (`aguardando_ponto → pronta → em_rota → entregue / devolvida`)
- Controle de status da expedição (`rascunho → pronta → despachada → concluída`)
- Visualização do painel de status para o lojista
- Listagem de entregas cadastradas na expedição em aberto (com nome, cidade, excursão, volumes, tamanho, prazo e entregador)

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Cria expedições, adiciona entregas, define pontos, acompanha status |
| **Sistema** | Atualiza status ao receber eventos de M2 (plano confirmado) e M3 (comprovante registrado) |

### Dependências

| Módulo | Motivo |
|---|---|
| M4 — Compradores | Busca o comprador ao criar uma entrega |
| M5 — Polo e Pontos | Consulta ponto e prazo ao definir destino da entrega |
| M2 — Roteirização | Recebe evento de plano confirmado para atualizar status das entregas para `em_rota` |
| M3 — Comprovante | Recebe evento de comprovante registrado para marcar entrega como `entregue` |

### Regras críticas

- Uma entrega pode existir sem ponto definido (`aguardando_ponto`) — não bloqueia o cadastro.
- Uma entrega não pode ir para `em_rota` sem ponto definido e sem um plano ativo.
- Uma entrega só é marcada como `entregue` após comprovante com assinatura registrado em M3.
- Uma entrega vai para `devolvida` somente por uma ocorrência registrada em M3.

---

## 3. M2 — Roteirização

### O que faz

Gera o plano de entrega ordenado para o entregador, considerando urgência de prazo e agrupamento geográfico. Permite confirmação pelo lojista, repriorização manual e acompanhamento da execução.

### O que pertence a este módulo

- Geração do plano de entrega (sugestão por IA determinística)
- Ordenação das entregas: urgência (prazo) → setor → vaga numérica
- Cálculo de nível de urgência por entrega (`crítico / atenção / ok`)
- Atribuição do entregador pelo lojista
- Confirmação do plano pelo lojista
- Repriorização manual de entregas no plano em andamento
- Envio do plano ao app do entregador
- Rastreamento de posição da execução (qual entrega está em andamento)

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Aciona geração do plano, confirma, atribui entregador, reprioriza |
| **Entregador** | Recebe o plano no app, solicita próxima entrega, confirma chegada na vaga |
| **Sistema** | Recalcula urgência em tempo real quando prazos mudam |

### Dependências

| Módulo | Motivo |
|---|---|
| M1 — Expedição | Lê as entregas prontas (com ponto definido) para gerar o plano |
| M5 — Polo e Pontos | Consulta setor (GPS), vaga e prazo de cada ponto |
| M6 — Identidade | Valida que o entregador atribuído está cadastrado na loja |

### Regras críticas

- O plano só inclui entregas com ponto definido.
- Urgência `crítico`: prazo efetivo < 30 minutos.
- Urgência `atenção`: prazo efetivo entre 30 e 60 minutos.
- Urgência `ok`: prazo efetivo > 60 minutos ou sem prazo definido.
- Repriorização manual altera a `ordemExecucao` sem recalcular urgência.
- O plano não pode ser alterado pelo entregador — somente pelo lojista.

---

## 4. M3 — Comprovante e Ocorrência

### O que faz

Registra provas de entrega (foto + assinatura + GPS) quando a entrega é bem-sucedida, e registra ocorrências com motivo quando não é — alimentando o controle de devoluções do lojista.

### O que pertence a este módulo

- Captura de foto da mercadoria no local
- Captura obrigatória da foto da assinatura de quem recebeu
- Registro das coordenadas GPS no momento da entrega
- Armazenamento seguro das fotos (storage externo)
- Registro de ocorrências com motivo (comprador ausente, recusou, ponto incorreto, outro)
- Visualização de devoluções no painel do lojista (com motivo)

### Atores

| Ator | Interação |
|---|---|
| **Entregador** | Captura fotos, registra comprovante ou ocorrência no campo via app |
| **Lojista** | Visualiza comprovantes e devoluções no painel |

### Dependências

| Módulo | Motivo |
|---|---|
| M1 — Expedição | Notifica M1 ao registrar comprovante (`entrega → entregue`) ou ocorrência (`entrega → devolvida`) |
| M7 — Notificação | `[Futuro]` Aciona notificação ao comprador após comprovante registrado |

### Regras críticas

- `fotoAssinatura` é obrigatória — o comprovante não é válido sem ela.
- O GPS é capturado automaticamente no momento do registro — não pode ser informado manualmente.
- A ocorrência só é válida com um motivo selecionado.
- Um comprovante ou ocorrência não pode ser editado após o registro.

---

## 5. M4 — Compradores

### O que faz

Mantém o cadastro de compradores da loja, serve como base para busca e desambiguação por nome, e guarda o ponto habitual para sugestão automática.

### O que pertence a este módulo

- Cadastro de compradores (nome, cidade, telefone opcional)
- Busca por nome (retorna lista para desambiguação quando há múltiplos com o mesmo nome)
- Ponto habitual por comprador (pré-cadastrado; sugerido automaticamente ao criar entrega)
- Histórico de entregas por comprador (referência)

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Cadastra e busca compradores |
| **Sistema** | Consulta ponto habitual ao criar uma nova entrega para o comprador |

### Dependências

| Módulo | Motivo |
|---|---|
| M5 — Polo e Pontos | Ponto habitual referencia um `PontoId` do M5 |

### Regras críticas

- `telefone` é opcional — a ausência não bloqueia nenhum fluxo.
- A busca por nome deve retornar múltiplos resultados quando houver ambiguidade — nunca selecionar automaticamente.
- O ponto habitual é uma sugestão, não um valor obrigatório — o lojista pode definir outro ponto na criação da entrega.

---

## 6. M5 — Polo e Pontos

### O que faz

Mantém o mapa canônico do polo: setores com coordenadas GPS, vagas por tipo de veículo e excursões (pontos de coleta) com seus horários de partida. É a fonte de verdade para endereçamento físico e cálculo de prazos.

### Estrutura de uma excursão

| Campo | Tipo | Observação |
|---|---|---|
| **Nome** | texto | Ex.: "Abdon Turismo", "Recife Moda" |
| **Tipo** | enum fixo | `Excursão` / `Transportadora` / `Individual` — valores fixos no banco, não cadastráveis |
| **Setor** | referência | Ex.: "laranja", "azul" — setor do polo onde o ponto fica |
| **Vaga** | texto | Ex.: "33" — vaga dentro do setor |
| **Tipo de veículo** | enum | Ex.: "pesado", "leve" |
| **Horário de saída** | hora | Partida do ponto; base para cálculo de prazo |

### Excursões do sistema vs. excursões da loja

| Origem | Quem cria | Visibilidade |
|---|---|---|
| **Sistema** | Administrador do polo | Disponível para todas as lojas |
| **Loja** | Lojista (via modal `+` na tela de entrega) | Privada — só aparece para a loja que criou |

### O que pertence a este módulo

- Cadastro de setores (cor, coordenadas GPS) — dados fixos do polo
- Excursões do sistema: pré-cadastradas pelo polo, visíveis a todas as lojas
- Excursões da loja: cadastradas pelo lojista, privadas da loja
- Horário de saída e cálculo de prazo efetivo por excursão
- Cálculo de prazo efetivo: `horarioSaida - tempoMedioFila`
- Atualização de ponto em campo pelo entregador (quando o comprador informa localização diferente)
- Listagem de excursões disponíveis para seleção no cadastro de entrega (sistema + da loja)

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Cadastra excursões específicas da loja; atualiza horários quando necessário |
| **Entregador** | Consulta ponto no app; pode corrigir localização em campo |
| **Sistema** | Consulta excursão e prazo ao gerar plano (M2) e ao definir ponto na entrega (M1) |

### Dependências

Nenhuma — M5 é o módulo base, consultado pelos demais. Consome Google Maps API externamente (via ACL) apenas para exibição de mapa.

### Regras críticas

- Os tipos de excursão (Excursão, Transportadora, Individual) são enum fixo no banco — o lojista seleciona, não cadastra tipos novos.
- Excursões do sistema são visíveis para todas as lojas; excursões da loja são privadas.
- O prazo efetivo é calculado pelo sistema — nunca informado manualmente.
- A correção de ponto em campo pelo entregador é registrada no histórico da excursão.
- Os setores e suas coordenadas GPS são dados fixos do polo (raramente mudam).
- O horário de saída pode ser atualizado pelo lojista quando o comprador avisa que vai sair antes.

---

## 7. M6 — Identidade e Acesso

### O que faz

Autentica lojistas e entregadores em um app único, com tela de login unificada. O sistema identifica o papel do usuário pelas credenciais e redireciona automaticamente para o painel correto. Conduz o lojista por um wizard de primeiro acesso que configura a loja, as coordenadas e o primeiro entregador antes de entrar no sistema.

> **Comprador não tem conta no sistema.** É apenas um registro de dados gerenciado pelo lojista. Nenhum acesso ao app.

---

### Login único — detecção automática de papel

Uma tela de login serve para lojistas e entregadores. O sistema detecta o papel pelas credenciais e persiste na sessão.

**Lojista** — pode entrar com qualquer uma das três formas:

| Forma | Exemplo | Observação |
|---|---|---|
| **Username** | `modafashion` | Escolhido no cadastro; único no sistema |
| **E-mail** | `loja@email.com` | Opcional no cadastro; só disponível se informado |
| **Código da loja** | `012312` | Gerado automaticamente no cadastro; fixo, sem expiração |

**Entregador** — entra com:

| Campo | Observação |
|---|---|
| **Username** | Único dentro da loja; credenciais separadas por loja (MVP) |
| **PIN** | Numérico de 6 dígitos; definido pelo lojista no cadastro |

**Senha:** PIN numérico de 6 dígitos para ambos os perfis.

**Sessão:** 15 dias com opção "manter conectado". Papel persiste na sessão — não é perguntado novamente.

---

### Lojista — hierarquia de usuários

Uma loja pode ter múltiplos usuários com dois níveis de permissão:

| Nível | Quem é | Permissões |
|---|---|---|
| **Master** | Criado no wizard (dono da loja) | Acesso total; único que pode deletar outros usuários |
| **Admin** | Cadastrado pelo master | Todas as funções do master, **exceto** deletar o master |

- O master não pode ser deletado por um admin.
- Admins podem cadastrar e remover entregadores, gerenciar expedições, rotas e pontos.
- Futuro: diferenciação mais granular de permissões por função.

---

### Lojista — wizard de primeiro acesso

Fluxo executado **uma única vez** no primeiro acesso. 3 passos. O usuário criado no wizard é automaticamente o **master** da loja.

| Passo | O que acontece |
|---|---|
| **1 — Dados da loja** | Nome da loja + e-mail + senha (PIN de 6 dígitos). Sistema gera o código da loja automaticamente. |
| **2 — Localização da loja** | Botão **"Usar minha localização agora"** — lê o GPS do celular, exibe o pin no mapa para o lojista confirmar. Fallback: busca por endereço. Feito uma única vez. |
| **3 — Primeiro entregador** | Cadastro de nome + login + senha temporária do entregador. Obrigatório antes de concluir. |
| **Conclusão** | Sistema exibe o **código de 6 dígitos** gerado — lojista anota/salva. Entra no painel. |

> Os setores do polo vêm **pré-mapeados** no sistema. Novos pontos podem ser adicionados pelo lojista nas configurações após o cadastro.

---

### Entregador — cadastro, login e segurança

Cadastrado pelo lojista (master ou admin) no wizard ou depois no painel.

| Campo | Obrigatório | Observação |
|---|---|---|
| **Nome** | Sim | Nome para exibição no painel do lojista |
| **Username** | Sim | Único dentro da loja |
| **PIN temporário** | Sim | Definido pelo lojista; deve ser trocado no primeiro acesso |
| **E-mail** | Não | Não solicitado |

**Primeiro acesso:** o sistema obriga o entregador a trocar o PIN antes de qualquer outra ação. Após a troca, nem o lojista tem acesso à senha do entregador.

**Recuperação de acesso:** o entregador solicita ao lojista, que gera um novo PIN temporário. O entregador deve trocá-lo no próximo acesso.

**Multi-loja (MVP):** um entregador pode trabalhar para mais de uma loja, mas com credenciais independentes em cada uma — cadastrado separadamente em cada loja com username e PIN distintos.

**Sessão:** mesma política do lojista — 15 dias com opção "manter conectado".

O entregador acessa apenas o app com o plano atribuído a ele. Não acessa o painel de gestão do lojista.

---

### Recuperação de acesso

| Perfil | Como recupera |
|---|---|
| **Lojista (master/admin)** | Link de redefinição enviado por e-mail |
| **Entregador** | Lojista gera PIN temporário; entregador troca no próximo acesso |

---

### O que pertence a este módulo

- Tela de login única com detecção automática de papel (lojista ou entregador)
- Cadastro e login do lojista (username / e-mail / código + PIN)
- Geração automática do código de 6 dígitos da loja
- Wizard de primeiro acesso — cria o usuário master da loja
- Hierarquia de usuários da loja: master e admin
- Cadastro e login do entregador (username + PIN, sem e-mail, credenciais por loja)
- Troca obrigatória de PIN no primeiro acesso do entregador
- Recuperação de acesso: e-mail para lojista; reset pelo lojista para entregador
- Gestão de sessão de 15 dias com opção "manter conectado"

### Atores

| Ator | Interação |
|---|---|
| **Lojista master** | Cadastra a loja, passa pelo wizard, gerencia usuários e entregadores |
| **Lojista admin** | Mesmas funções do master, exceto deletar o master |
| **Entregador** | Faz login com username + PIN; troca senha no primeiro acesso |

### Dependências

| Módulo | Motivo |
|---|---|
| M5 — Polo e Pontos | Passo 3 do wizard cadastra novos pontos diretamente em M5 |

Os demais módulos recebem o `UsuárioAutenticado` já resolvido — não dependem de M6 diretamente.

### Regras críticas

- O sistema detecta o papel do usuário pelas credenciais — não há seleção manual de papel na tela de login.
- Um entregador só acessa os dados da loja que o cadastrou.
- O lojista só acessa as expedições e dados da sua própria loja.
- O master não pode ser deletado por um admin.
- O entregador é obrigado a trocar o PIN no primeiro acesso — o fluxo do app é bloqueado até a troca ser feita.
- O código de 6 dígitos da loja não expira e não muda (a menos que o lojista solicite regeneração futuramente).
- O wizard de primeiro acesso (3 passos) deve ser concluído antes de o lojista acessar o painel — nenhum passo pode ser pulado.
- A localização da loja é definida uma única vez no wizard; pode ser editada nas configurações depois.
- Credenciais do entregador são por loja no MVP — conta unificada multi-loja é evolução futura.

---

## 8. M7 — Notificação `[Futuro]`

### O que faz

Dispara automaticamente mensagem ao comprador via WhatsApp ou SMS quando uma entrega é confirmada (comprovante registrado).

### O que pertence a este módulo

- Envio de notificação de entrega concluída ao comprador
- Canal preferencial: WhatsApp (se número cadastrado)
- Fallback: SMS (se não tiver WhatsApp)
- Sem envio se `telefone` do comprador estiver vazio

### Atores

| Ator | Interação |
|---|---|
| **Sistema** | Aciona ao receber evento `EntregaConcluída` de M3 |
| **Comprador** | Recebe mensagem passivamente (não usa o sistema) |

### Dependências

| Módulo | Motivo |
|---|---|
| M3 — Comprovante | Evento `EntregaConcluída` como gatilho |
| M4 — Compradores | Busca `telefone` do comprador para envio |

### Regras críticas

- A ausência de `telefone` não causa erro — a notificação simplesmente não é enviada.
- O envio é assíncrono — não bloqueia o fluxo de registro do comprovante.

---

## 9. Mapa de Dependências entre Módulos

```
M6 — Identidade
  └── todos os módulos recebem UsuárioAutenticado (nenhum depende de M6 diretamente)

M5 — Polo e Pontos  ◄── consultado por M1, M2 e M4
  └── consome Google Maps API (ACL externa)

M4 — Compradores  ◄── consultado por M1 e M7
  └── referencia PontoId de M5

M1 — Expedição
  ├── consulta M4 (comprador) e M5 (ponto/prazo)
  ├── recebe eventos de M2 (plano confirmado → em_rota)
  └── recebe eventos de M3 (comprovante → entregue / ocorrência → devolvida)

M2 — Roteirização
  ├── lê entregas prontas de M1
  ├── consulta setor/prazo de M5
  └── emite evento PlanoConfirmado → M1

M3 — Comprovante e Ocorrência
  ├── emite evento EntregaConcluída → M1
  └── emite evento EntregaConcluída → M7 [Futuro]

M7 — Notificação [Futuro]
  ├── consome evento de M3
  └── consulta telefone de M4
```

Nenhum ciclo de dependência. M5 é o módulo base; M6 é transversal (autenticação).

---

## 10. Escopo por Módulo — MVP vs. Futuro

| Módulo | MVP | Futuro |
|---|---|---|
| **M1 — Expedição** | Criação manual com autocomplete (cliente, cidade, excursão); estados completos; painel de acompanhamento | Criação por voz (wizard IA — usa o mesmo índice do autocomplete); criação por foto/OCR; integração com soft do lojista |
| **M2 — Roteirização** | Geração determinística por urgência + setor + vaga; atribuição manual do entregador; repriorização manual | IA para sugestão de prioridades entre entregas; repriorização automática; atribuição automática por carga/disponibilidade; marketplace de entregadores avulsos |
| **M3 — Comprovante** | Foto mercadoria + foto assinatura + GPS; ocorrência com motivo; listagem de devoluções | — |
| **M4 — Compradores** | Cadastro; busca com desambiguação; ponto habitual | Histórico analítico por comprador |
| **M5 — Polo e Pontos** | Setores e pontos de Santa Cruz do Capibaribe; horários de partida; correção em campo | Expansão para outros polos; mapa 3D gamificado com countdown de prazo |
| **M6 — Identidade** | Login lojista + entregador; cadastro de entregador pela loja | — |
| **M7 — Notificação** | — | WhatsApp/SMS ao comprador após entrega confirmada |

---

*Próxima etapa: [Etapa 3 — Requisitos Funcionais e Não Funcionais](requisitos.md)*
