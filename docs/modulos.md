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

### O que pertence a este módulo

- Criação de expedições com uma ou mais entregas
- Associação de comprador e volumes a cada entrega
- Definição ou atualização do ponto de destino de uma entrega
- Controle de status de cada entrega (`aguardando_ponto → pronta → em_rota → entregue / devolvida`)
- Controle de status da expedição (`rascunho → pronta → despachada → concluída`)
- Visualização do painel de status para o lojista

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

Mantém o mapa canônico do polo: setores com coordenadas GPS, vagas por tipo de veículo, pontos (excursões e compradores independentes) e seus horários de partida. É a fonte de verdade para endereçamento físico e cálculo de prazos.

### O que pertence a este módulo

- Cadastro de setores (cor, coordenadas GPS)
- Cadastro de pontos com localização (setor + vaga + tipo de veículo)
- Horário de partida e tempo médio de fila por ponto
- Cálculo de prazo efetivo: `horarioPartida - tempoMedioFila`
- Atualização de ponto em campo pelo entregador (quando o comprador informa localização diferente)
- Listagem de pontos para seleção no cadastro de entrega

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Cadastra e mantém pontos e horários de excursões |
| **Entregador** | Consulta ponto no app; pode corrigir localização em campo |
| **Sistema** | Consulta ponto e prazo ao gerar plano (M2) e ao definir ponto na entrega (M1) |

### Dependências

Nenhuma — M5 é o módulo base, consultado pelos demais. Consome Google Maps API externamente (via ACL) apenas para exibição de mapa.

### Regras críticas

- O prazo efetivo é calculado pelo sistema — nunca informado manualmente.
- A correção de ponto em campo pelo entregador é registrada no histórico do ponto.
- Os setores e suas coordenadas GPS são dados fixos do polo (raramente mudam).
- O horário de partida é por ponto/excursão — pode ser atualizado pelo lojista quando o comprador avisa que vai sair antes.

---

## 7. M6 — Identidade e Acesso

### O que faz

Autentica lojistas e entregadores com fluxos simples e rápidos, pensados para usuários com baixa familiaridade com tecnologia. Conduz o lojista por um wizard de primeiro acesso que configura a loja, as coordenadas e o primeiro entregador antes de entrar no sistema.

---

### Lojista — login

O lojista pode entrar com qualquer uma das três formas:

| Forma | Exemplo | Observação |
|---|---|---|
| **Username** | `modafashion` | Escolhido no cadastro; único no sistema |
| **E-mail** | `loja@email.com` | Opcional no cadastro; só disponível se informado |
| **Código da loja** | `012312` | Gerado automaticamente no cadastro; fixo, sem expiração |

**Senha:** PIN numérico de 6 dígitos (ex: `123456`). Simples e rápido de digitar no celular.

**Sessão:** 15 dias. O lojista pode salvar a sessão e não precisa fazer login a cada acesso.

---

### Lojista — wizard de primeiro acesso

Fluxo executado **uma única vez** no primeiro acesso. 3 passos. Objetivo: deixar a loja pronta para operar.

| Passo | O que acontece |
|---|---|
| **1 — Dados da loja** | Nome da loja + e-mail + senha (PIN de 6 dígitos). Sistema gera o código da loja automaticamente. |
| **2 — Localização da loja** | Botão **"Usar minha localização agora"** — lê o GPS do celular, exibe o pin no mapa para o lojista confirmar. Fallback: busca por endereço. Feito uma única vez. |
| **3 — Primeiro entregador** | Cadastro de nome + login + senha do entregador. Obrigatório antes de concluir. |
| **Conclusão** | Sistema exibe o **código de 6 dígitos** gerado — lojista anota/salva. Entra no painel. |

> Os setores do polo vêm **pré-mapeados** no sistema. Novos pontos podem ser adicionados pelo lojista nas configurações após o cadastro.

---

### Entregador — login

Cadastrado pelo lojista (no wizard ou depois, no painel).

| Campo | Obrigatório | Observação |
|---|---|---|
| **Nome** | Sim | Nome para exibição no painel do lojista |
| **Login** | Sim | Username simples, único dentro da loja |
| **PIN** | Sim | Numérico de 6 dígitos |
| **E-mail** | Não | Não solicitado |

**Sessão:** mesma política do lojista — 15 dias, com opção de salvar.

O entregador acessa apenas o app mobile com o plano atribuído a ele. Não acessa o painel de gestão do lojista.

---

### O que pertence a este módulo

- Cadastro e login do lojista (username / e-mail / código + PIN)
- Geração automática do código de 6 dígitos da loja
- Wizard de primeiro acesso (localização da loja + pontos adicionais + primeiro entregador)
- Cadastro e login do entregador (login + PIN, sem e-mail)
- Gestão de sessão de 15 dias com opção "manter conectado"
- Separação de permissões: lojista acessa painel completo da sua loja; entregador acessa apenas seu plano ativo

### Atores

| Ator | Interação |
|---|---|
| **Lojista** | Cadastra a loja, passa pelo wizard, faz login pelas três formas |
| **Entregador** | Faz login com username + PIN no app mobile |

### Dependências

| Módulo | Motivo |
|---|---|
| M5 — Polo e Pontos | Passo 5 do wizard cadastra novos pontos diretamente em M5 |

Os demais módulos recebem o `UsuárioAutenticado` já resolvido — não dependem de M6 diretamente.

### Regras críticas

- Um entregador só acessa os dados da loja que o cadastrou.
- O lojista só acessa as expedições e dados da sua própria loja.
- O código de 6 dígitos da loja não expira e não muda (a menos que o lojista solicite regeneração futuramente).
- O wizard de primeiro acesso (3 passos) deve ser concluído antes de o lojista acessar o painel — nenhum passo pode ser pulado.
- A localização da loja é definida uma única vez no wizard; pode ser editada nas configurações depois.

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
| **M1 — Expedição** | Criação manual; estados completos; painel de acompanhamento | Criação por voz (wizard IA); criação por foto/OCR; integração com soft do lojista |
| **M2 — Roteirização** | Geração determinística por urgência + setor + vaga; atribuição manual do entregador; repriorização manual | Repriorização automática por IA; atribuição automática por carga/disponibilidade; marketplace de entregadores avulsos |
| **M3 — Comprovante** | Foto mercadoria + foto assinatura + GPS; ocorrência com motivo; listagem de devoluções | — |
| **M4 — Compradores** | Cadastro; busca com desambiguação; ponto habitual | Histórico analítico por comprador |
| **M5 — Polo e Pontos** | Setores e pontos de Santa Cruz do Capibaribe; horários de partida; correção em campo | Expansão para outros polos; mapa 3D gamificado com countdown de prazo |
| **M6 — Identidade** | Login lojista + entregador; cadastro de entregador pela loja | — |
| **M7 — Notificação** | — | WhatsApp/SMS ao comprador após entrega confirmada |

---

*Próxima etapa: [Etapa 3 — Requisitos Funcionais e Não Funcionais](requisitos.md)*
