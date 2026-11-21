# Glossário e Modelagem de Domínio — Exped

Produzido com o framework Domain-Driven Design como referência de vocabulário, contextos e eventos do domínio. A estrutura de código do projeto segue **Clean Architecture** — ver [arquitetura.md](arquitetura.md).

> Este documento serve como referência de linguagem ubíqua, bounded contexts e domain events. Não define camadas de código nem padrões de implementação.

---

## Índice

1. [Linguagem Ubíqua](#1-linguagem-ubíqua)
2. [Domínio Central e Subdomínios](#2-domínio-central-e-subdomínios)
3. [Bounded Contexts](#3-bounded-contexts)
4. [Mapa de Contextos (Context Map)](#4-mapa-de-contextos-context-map)
5. [Aggregates, Entities e Value Objects](#5-aggregates-entities-e-value-objects)
6. [Domínio de Eventos](#6-domínio-de-eventos)
7. [Repositórios](#7-repositórios)
8. [Score do Modelo](#8-score-do-modelo)

---

## 1. Linguagem Ubíqua

Derivada do glossário do discovery. Todos os termos abaixo devem aparecer sem variação em código, banco de dados, documentação e conversa.

| Termo | Definição de domínio |
|---|---|
| **Expedição** | Registro de saída criado pelo lojista — agrupa 1 ou mais entregas individuais para diferentes compradores. Pode existir sem entregador atribuído. |
| **Entrega** | Unidade individual dentro de uma expedição — um ou mais volumes destinados a um comprador específico, a um ponto. Pode existir sem ponto definido (aguardando ponto). |
| **Ponto** | Entidade registrada no sistema que representa onde um comprador aguarda os volumes: setor + vaga + tipo de veículo. Pode ser uma excursão ou um comprador independente. |
| **Setor** | Área do polo identificada por cor (Laranja, Verde, Branco, Azul, Vermelho, Amarelo) com coordenadas GPS. Unidade primária de agrupamento para roteirização. |
| **Vaga** | Posição numerada dentro de um setor onde o veículo do comprador estaciona. |
| **Excursão** | Tipo de ponto — van ou ônibus com múltiplos compradores de outra cidade. Possui horário de partida que define o prazo-limite de todas as entregas vinculadas. |
| **CompradorIndependente** | Tipo de ponto — comprador que chega com veículo próprio. Pode ou não ter horário de saída definido. |
| **HorárioDePartida** | Hora prevista em que o ponto deixa o polo. Define o prazo máximo de entrega. |
| **PrazoDeEntrega** | Derivado do HorárioDePartida do ponto, descontado o tempo médio de fila. Usado pelo sistema para priorizar o roteiro. |
| **PlanoDeEntrega** | Conjunto ordenado de entregas para o entregador executar numa saída. Gerado pela IA no momento do despacho. |
| **Comprovante** | Registro obrigatório feito pelo entregador ao concluir uma entrega: foto da mercadoria + foto da assinatura + GPS. |
| **Ocorrência** | Registro de uma entrega que não pôde ser concluída — com motivo. O volume entra em status devolvido. |
| **Devolução** | Volume que retornou à loja após tentativa malsucedida. Visível no painel do lojista com motivo registrado. |
| **Repriorização** | Alteração manual da ordem do roteiro em andamento pelo lojista. |
| **Lojista** | Dono ou operador da loja. Cria expedições, aprova o plano de entrega, monitora status. |
| **Entregador** | Pessoa cadastrada que executa as entregas. Usa o app mobile no campo. |
| **Comprador** | Cliente do lojista. Não usa o sistema diretamente. É o destinatário final das entregas. |
| **Volume** | Pacote físico individual (caixa, saco etc.). Uma entrega pode ter múltiplos volumes. |
| **WizardDeIA** | Interface conversacional (voz ou texto) que conduz o lojista pela criação de uma expedição. `[Futuro]` |
| **Soft** | Sistema externo já existente do lojista para registro de compras. |

---

## 2. Domínio Central e Subdomínios

### Domínio Central (Core Domain) — investir modelagem profunda

**Expedição + Roteirização com Prazo**

É onde o diferencial competitivo do produto vive:
- Criação rápida de expedições (voz, OCR, manual)
- Plano de entrega gerado pela IA com prioridade por prazo e agrupamento geográfico
- Repriorização em tempo real quando os prazos mudam

Nenhum chatbot genérico ou app concorrente combina as três camadas simultaneamente. É aqui que os melhores recursos de engenharia devem ser aplicados.

### Subdomínios de Suporte (Supporting) — construir, sem over-engineering

| Subdomínio | Responsabilidade |
|---|---|
| **Polo Geográfico** | Mapa de setores, vagas, pontos, coordenadas GPS, excursões e horários |
| **Comprovante de Entrega** | Captura de foto, assinatura e GPS stamp em campo |
| **Gestão de Ocorrências** | Registro de falhas de entrega, devoluções e status resultante |
| **Catálogo de Compradores** | Histórico de compradores, pontos habituais, desambiguação por nome |

### Subdomínios Genéricos (Generic) — comprar ou usar serviço externo

| Subdomínio | Decisão |
|---|---|
| **Autenticação / Identidade** | Usar solução de mercado (Auth0, Supabase Auth, Firebase Auth etc.) |
| **Mapas e Roteirização GPS** | Google Maps API (MVP); OpenStreetMap como alternativa de custo `[Avaliar]` |
| **Notificações WhatsApp/SMS** | Twilio / Meta Cloud API `[Futuro]` |
| **Armazenamento de Fotos** | Object storage (S3, Cloudflare R2 etc.) |

---

## 3. Bounded Contexts

### BC1 — Expedição `[Core]`

**Responsabilidade:** Ciclo de vida completo de uma expedição — criação, associação de ponto, confirmação, despacho, conclusão ou devolução.

Conceitos que vivem aqui: `Expedição`, `Entrega`, `Volume`, `StatusEntrega`, `Ocorrência`, `Devolução`.

Estados de uma `Entrega`:
```
aguardando_ponto → pronta → em_rota → entregue
                                    ↘ ocorrencia → devolvida
```

Estados de uma `Expedição`:
```
rascunho → pronta → despachada → concluida (todas as entregas finalizadas)
```

---

### BC2 — Roteirização `[Core]`

**Responsabilidade:** Gerar e gerenciar PlanoDeEntrega — ordenação de entregas por prazo urgência (HorárioDePartida) e agrupamento geográfico (Setor → Vaga).

Conceitos que vivem aqui: `PlanoDeEntrega`, `OrdemDeEntrega`, `PrazoDeEntrega`, `UrgênciaDePrazo`.

Este contexto **não possui dados de ponto** — ele consome a localização e o prazo via integração com BC3 (Polo Geográfico).

---

### BC3 — Polo Geográfico `[Supporting]`

**Responsabilidade:** Mapa canônico do polo — setores com GPS, vagas, pontos, excursões e seus horários de partida. Fonte de verdade para endereçamento físico e cálculo de prazos.

Conceitos que vivem aqui: `Polo`, `Setor`, `Vaga`, `Ponto`, `Excursão`, `CompradorIndependente`, `HorárioDePartida`, `TipoVeiculo`, `TempoMédioFila`.

---

### BC4 — Comprovante `[Supporting]`

**Responsabilidade:** Captura e armazenamento de provas de entrega. Totalmente desacoplado do fluxo de expedição — é acionado pelo entregador no campo.

Conceitos que vivem aqui: `Comprovante`, `FotoMercadoria`, `FotoAssinatura`, `CoordenadasGPS`, `MomentoDeRegistro`.

---

### BC5 — Catálogo de Compradores `[Supporting]`

**Responsabilidade:** Cadastro e histórico de compradores — base para desambiguação por nome e sugestão automática de ponto habitual.

Conceitos que vivem aqui: `Comprador`, `PontoHabitual`, `HistóricoDeEntregas` (referência).

---

### BC6 — Identidade e Acesso `[Generic]`

**Responsabilidade:** Autenticação e perfis de Lojista e Entregador. Sem modelagem de domínio rica — usar solução externa.

Conceitos que vivem aqui: `Lojista`, `Entregador`, `Loja`, `Credenciais`.

---

### BC7 — Notificação `[Generic — Futuro]`

**Responsabilidade:** Disparar mensagens ao Comprador via WhatsApp ou SMS após confirmação de entrega.

Conceitos que vivem aqui: `Notificação`, `CanalDeEnvio`, `DestinatárioNotificação`.

---

## 4. Mapa de Contextos (Context Map)

```
                        ┌─────────────────────┐
                        │  BC6 · Identidade   │
                        │  (Generic / Externo) │
                        └──────────┬──────────┘
                     autentica     │
         ┌───────────────────────── ┤
         │                         │
         ▼                         ▼
┌────────────────┐        ┌────────────────────┐
│  BC1           │        │  BC2               │
│  Expedição     │◄──────►│  Roteirização      │
│  [Core]        │        │  [Core]            │
└───────┬────────┘        └────────┬───────────┘
        │                          │
        │  consome ponto/prazo     │  consome setor/GPS/prazo
        ▼                          ▼
┌───────────────────────────────────────┐
│          BC3 · Polo Geográfico         │
│          [Supporting]                  │
└───────────────────────────────────────┘
        │
        │ ACL — Google Maps API
        ▼
   [Google Maps / OpenStreetMap]

BC1 · Expedição ──── ao concluir entrega ────► BC4 · Comprovante
BC5 · Compradores ─── sugere ponto habitual ──► BC1 · Expedição
BC1 · Expedição ──── ao confirmar entrega ────► BC7 · Notificação [Futuro]
```

**Padrões de integração por fronteira:**

| Fronteira | Padrão | Motivo |
|---|---|---|
| BC1 ↔ BC2 | Customer/Supplier | Expedição gera dados; Roteirização consome para gerar plano |
| BC1 ↔ BC3 | Customer/Supplier + ACL | Expedição consome Ponto e Prazo do Polo; ACL protege o modelo |
| BC2 ↔ BC3 | Customer/Supplier | Roteirização consome GPS dos setores |
| BC1 → BC4 | Published Event | `EntregaConcluída` dispara criação de Comprovante |
| BC5 → BC1 | Open Host Service | Compradores expõem sugestão de ponto via API interna |
| BC1 → BC7 | Published Event + ACL | `ComprovanteCriado` dispara notificação via adapter WhatsApp/SMS |
| BC2 → Google Maps | ACL | Nunca deixar o modelo do Maps vazar para dentro do domínio |
| BC6 → todos | Conformist | Todos os contextos aceitam identidade do serviço externo |

---

## 5. Aggregates, Entities e Value Objects

### BC1 — Expedição

#### Aggregate: `Expedição`

**Root:** `Expedição`

```
Expedição (Aggregate Root)
  - id: ExpediçãoId
  - lojista: LojistId (referência por ID para BC6)
  - status: StatusExpedição  [VO — enum: rascunho | pronta | despachada | concluida]
  - criadaEm: Momento        [VO]
  - entregas: List<Entrega>  [Entity interna — só existe através de Expedição]
```

**Entity interna:** `Entrega`

```
Entrega (Entity)
  - id: EntregaId
  - comprador: CompradorId   (referência por ID para BC5)
  - ponto: PontoId?          (referência por ID para BC3 — nullable até chegar o comprador)
  - volumes: List<Volume>    [VO]
  - status: StatusEntrega    [VO — enum: aguardando_ponto | pronta | em_rota | entregue | ocorrencia | devolvida]
  - observacao: Texto?       [VO]
  - prazo: PrazoDeEntrega?   [VO — calculado a partir do Ponto]
```

**Value Objects:**

| VO | Atributos | Invariante |
|---|---|---|
| `Volume` | quantidade: Int, tamanho: TamanhoVolume (P/M/G) | quantidade >= 1 |
| `StatusEntrega` | enum | transições válidas controladas pelo aggregate |
| `StatusExpedição` | enum | transições válidas controladas pelo aggregate |
| `PrazoDeEntrega` | horarioPartida: Hora, tempoFila: Minutos, prazoEfetivo: Hora | prazoEfetivo = horarioPartida - tempoFila |

**Invariantes do aggregate:**

- Uma `Expedição` precisa ter ao menos uma `Entrega`.
- `Entrega` só pode ir para `em_rota` se tiver `ponto` definido e um `PlanoDeEntrega` que a inclua.
- `Entrega` só pode ir para `entregue` se houver um `Comprovante` associado (via ID em BC4).
- `Entrega` vai para `devolvida` somente por uma `Ocorrência`.

---

#### Aggregate: `Ocorrência`

**Root:** `Ocorrência` (separado de Expedição para não inflar o aggregate raiz)

```
Ocorrência (Aggregate Root)
  - id: OcorrênciaId
  - entregaId: EntregaId   (referência por ID)
  - motivo: MotivoOcorrência  [VO — enum: comprador_ausente | recusou | ponto_incorreto | outro]
  - descricao: Texto?
  - registradaEm: Momento
  - registradaPor: EntregadorId
```

---

### BC2 — Roteirização

#### Aggregate: `PlanoDeEntrega`

**Root:** `PlanoDeEntrega`

```
PlanoDeEntrega (Aggregate Root)
  - id: PlanoId
  - expedicaoId: ExpediçãoId
  - entregador: EntregadorId
  - itens: List<ItemDePlano>  [VO]
  - status: StatusPlano       [VO — enum: sugerido | confirmado | em_execucao | concluido]
  - criadoEm: Momento
  - confirmadoEm: Momento?
```

**Value Object:** `ItemDePlano`

```
ItemDePlano [VO]
  - entregaId: EntregaId
  - ordemExecucao: Int
  - urgencia: NívelUrgência   [enum: critico | atencao | ok]
  - prazoEfetivo: PrazoDeEntrega
  - setorId: SetorId
  - vaga: NúmeroVaga
```

**Lógica de ordenação (Core Domain logic):**
1. Prioridade primária: `urgencia` = `critico` primeiro (prazo < 30min)
2. Prioridade secundária: agrupamento por `setorId` (minimiza deslocamento)
3. Prioridade terciária: `vaga` numérica dentro do setor

---

### BC3 — Polo Geográfico

#### Aggregate: `Ponto`

**Root:** `Ponto` (supertipo abstrato com dois subtipos)

```
Ponto (Aggregate Root)
  - id: PontoId
  - nome: NomePonto
  - localizacao: LocalizaçãoPonto  [VO]
  - tipo: TipoPonto                [VO — enum: excursao | comprador_independente]
  - horarioPartida: Hora?          (nulo para compradores sem horário definido)
  - tempoMedioFilaMinutos: Int
```

**Value Object:** `LocalizaçãoPonto`

```
LocalizaçãoPonto [VO — imutável]
  - setorId: SetorId
  - vaga: NúmeroVaga
  - tipoVeiculo: TipoVeiculo  [enum: van | pesado | gerador]
```

#### Aggregate: `Setor`

```
Setor (Aggregate Root)
  - id: SetorId
  - nome: NomeSetor
  - cor: CorSetor              [VO — enum: laranja | verde | branco | azul | vermelho | amarelo]
  - coordenadas: CoordenadasGPS  [VO]
```

**Value Object:** `CoordenadasGPS`

```
CoordenadasGPS [VO]
  - lat: Decimal
  - lng: Decimal
```

---

### BC4 — Comprovante

#### Aggregate: `Comprovante`

**Root:** `Comprovante`

```
Comprovante (Aggregate Root)
  - id: ComprovanteId
  - entregaId: EntregaId
  - entregadorId: EntregadorId
  - fotoMercadoria: URLFoto
  - fotoAssinatura: URLFoto     (obrigatório — segurança jurídica)
  - coordenadas: CoordenadasGPS
  - registradoEm: Momento
```

**Invariante:** `fotoAssinatura` é obrigatória — uma `Entrega` não pode ser marcada como `entregue` sem ela.

---

### BC5 — Catálogo de Compradores

#### Aggregate: `Comprador`

```
Comprador (Aggregate Root)
  - id: CompradorId
  - nome: NomeComprador
  - cidade: Cidade?
  - telefone: Telefone?        (opcional — sem número, notificação não é disparada)
  - pontoHabitual: PontoId?    (pré-cadastrado para sugestão automática)
```

**Invariante:** A ausência de `telefone` não impede o fluxo de entrega.

---

## 6. Domínio de Eventos

| Evento | Contexto de origem | Reações |
|---|---|---|
| `ExpediçãoCriada` | BC1 | Nenhuma reação imediata — expedição aguarda ponto e plano |
| `PontoDefinidoNaEntrega` | BC1 | BC2 recalcula urgência do plano se já existir |
| `PlanoDeEntregaConfirmado` | BC2 | BC1 marca entregas como `em_rota`; entregador recebe no app |
| `ChegadaNaVagaRegistrada` | BC2 | Lojista é notificado (push) |
| `EntregaConcluída` | BC1 | BC4 cria Comprovante; BC7 `[Futuro]` dispara notificação ao Comprador |
| `OcorrênciaRegistrada` | BC1 | BC1 marca Entrega como `devolvida`; Lojista vê no painel |
| `PrazoCríticoDetectado` | BC2 | Alerta visual no painel do lojista; `[Futuro]` repriorização automática |
| `ComprovanteArmazenado` | BC4 | BC1 recebe confirmação e finaliza a Entrega |

---

## 7. Repositórios

Interfaces vivem no domínio; implementações vivem na infraestrutura.

| Repositório | Métodos principais |
|---|---|
| `ExpediçãoRepository` | `save(expedição)`, `findById(id)`, `findPendentesByLojista(lojistId)` |
| `EntregaRepository` | `findAguardandoPonto(lojistId)`, `findByExpedição(expediçãoId)` |
| `PlanoDeEntregaRepository` | `save(plano)`, `findAtivoPorEntregador(entregadorId)` |
| `PontoRepository` | `findBySetor(setorId)`, `findById(id)`, `findAll()` |
| `SetorRepository` | `findAll()`, `findById(id)` |
| `CompradorRepository` | `findByNome(nome)` (retorna lista para desambiguação), `findById(id)` |
| `ComprovanteRepository` | `findByEntrega(entregaId)`, `save(comprovante)` |

---

## 8. Score do Modelo

Avaliação pelo Quick Diagnostic do Framework DDD:

| Critério | Status | Observação |
|---|---|---|
| Nomes legíveis por especialistas de domínio | ✅ | Expedição, Entrega, Ponto, Setor, Vaga, Comprovante — direto do glossário do lojista |
| Bounded contexts explicitamente definidos | ✅ | 7 contextos com responsabilidades isoladas e mapa de integração |
| Aggregates pequenos (1 root + cluster mínimo) | ✅ | Maior aggregate tem 2 entities (Expedição + Entrega) — referências por ID para o resto |
| Domain objects com comportamento, não só dados | ✅ | Invariantes, transições de estado e cálculo de prazo encapsulados |
| Domain events para comunicação cross-aggregate | ✅ | 8 eventos identificados com contexto de origem e reações |
| Anti-Corruption Layer em integrações externas | ✅ | Google Maps e WhatsApp/SMS isolados via ACL |
| Core Domain identificado | ✅ | Expedição + Roteirização com Prazo — foco claro do produto |
| Core Domain com modelo rico (não CRUD) | ✅ | Lógica de urgência, ordenação, transições e invariantes de prazo |
| Invariantes dentro dos aggregates | ✅ | Comprovante obrigatório, Entrega sem ponto não sai para rota |
| Linguagem ubíqua consistente em todo o documento | ✅ | Mesmo vocabulário do discovery sem variações |

**Score: 10/10**

---

*Próximo passo: [arquitetura.md](arquitetura.md) — decisões técnicas, stack e diagramas de infraestrutura.*
