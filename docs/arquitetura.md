# Arquitetura de Sistema — Exped (Clean Architecture)

Produzido com o framework Clean Architecture a partir do [discovery.md](discovery.md) e do glossário de domínio em [ddd.md](ddd.md).

---

## Índice

1. [Princípio Central](#1-princípio-central)
2. [Camadas do Sistema](#2-camadas-do-sistema)
3. [Mapa de Dependências](#3-mapa-de-dependências)
4. [Inventário de Use Cases](#4-inventário-de-use-cases)
5. [Estrutura de Pastas](#5-estrutura-de-pastas)
6. [Interfaces de Repositório](#6-interfaces-de-repositório)
7. [Decisões de Infraestrutura](#7-decisões-de-infraestrutura)
8. [Stack Tecnológico Proposto](#8-stack-tecnológico-proposto)
9. [Fluxos de Dados — Exemplo End-to-End](#9-fluxos-de-dados--exemplo-end-to-end)
10. [Estratégia de Deploy](#10-estratégia-de-deploy)
11. [Score do Modelo](#11-score-do-modelo)

---

## 1. Princípio Central

**Dependências de código-fonte apontam sempre para dentro — em direção às regras de negócio.**

```
Frameworks & Drivers
  └── Interface Adapters
        └── Use Cases
              └── Entities  ← nada depende de nada externo aqui
```

As regras de negócio do Exped (calcular prazo, ordenar rota por urgência, validar comprovante) não sabem que existem banco de dados, HTTP, Google Maps ou WhatsApp. Essas são todas *implementações detalhes* que podem ser trocadas sem tocar no núcleo.

---

## 2. Camadas do Sistema

### Camada 1 — Entities (Regras de Negócio Empresariais)

Objetos de domínio puros — sem dependências externas. Existiriam mesmo sem software.

| Entity | Regras encapsuladas |
|---|---|
| `Expedição` | Transições de status válidas; requer ao menos uma Entrega |
| `Entrega` | Pode existir sem ponto (estado `aguardando_ponto`); só vai para `em_rota` com ponto definido e plano ativo |
| `Ponto` | Encapsula `LocalizaçãoPonto` (setor + vaga + tipo_veículo); calcula `PrazoDeEntrega` a partir de `horarioPartida - tempoMedioFila` |
| `PlanoDeEntrega` | Lógica de ordenação: urgência (prazo) → setor → vaga numérica |
| `Comprovante` | `fotoAssinatura` é obrigatória — invariante que não pode ser violada |
| `Comprador` | `telefone` é opcional; `pontoHabitual` é sugerido automaticamente |

**Value Objects (imutáveis, sem identidade):**
`LocalizaçãoPonto`, `PrazoDeEntrega`, `NívelUrgência`, `CoordenadasGPS`, `Volume`, `StatusEntrega`, `StatusExpedição`

---

### Camada 2 — Use Cases (Regras de Negócio da Aplicação)

Cada use case é uma única operação do sistema. Recebe um `Request` simples, retorna um `Response` simples — nunca objetos do framework ou do ORM.

Padrão: `NomeUseCase.interactor.ts` implementa `NomeUseCaseInput`, chama `NomeUseCaseOutput`.

Ver inventário completo na [Seção 4](#4-inventário-de-use-cases).

---

### Camada 3 — Interface Adapters

Converte dados entre o formato conveniente para Use Cases/Entities e o formato exigido pelo mundo externo.

| Tipo | Responsabilidade |
|---|---|
| **Controller** | Recebe HTTP request → constrói Request Model → chama Use Case |
| **Presenter** | Recebe Response Model do Use Case → formata para JSON/view |
| **Repository (Gateway)** | Implementa interface de repositório definida nos Use Cases usando banco real |
| **Maps Gateway** | Adapta Google Maps API para a linguagem do domínio (`calcularRota(origem, destinos)`) |
| **Notificação Gateway** | Adapta Twilio/WhatsApp API para `notificarComprador(telefone, mensagem)` |
| **Storage Gateway** | Adapta S3/R2 para `armazenarFoto(arquivo) → url` |

---

### Camada 4 — Frameworks & Drivers

Código de colagem (glue code) com o mundo externo. O framework chama para dentro — nunca o contrário.

| Componente | Papel |
|---|---|
| **Web Framework** | Roteamento HTTP; chama Controllers |
| **ORM / Driver SQL** | Implementa os Repositories; confinado nesta camada |
| **Google Maps SDK** | Consumido exclusivamente pelo Maps Gateway |
| **Supabase / PostgreSQL** | Banco de dados; invisível para Use Cases |
| **Supabase Auth** | Autenticação; resultado chega aos Controllers como `UsuárioAutenticado` |
| **Cloudflare R2 / S3** | Armazenamento de fotos de comprovante |
| **Twilio / Meta API** | Envio de WhatsApp/SMS `[Futuro]` |

---

## 3. Mapa de Dependências

```
┌──────────────────────────────────────────────────────────────────────┐
│  FRAMEWORKS & DRIVERS (camada 4)                                     │
│                                                                      │
│  Express/Fastify  PostgreSQL  Google Maps SDK  Supabase  R2/S3       │
│        │               │            │              │        │        │
└────────┼───────────────┼────────────┼──────────────┼────────┼────────┘
         ▼               ▼            ▼              ▼        ▼
┌──────────────────────────────────────────────────────────────────────┐
│  INTERFACE ADAPTERS (camada 3)                                       │
│                                                                      │
│  ExpediçãoController     PostgresExpediçãoRepository                 │
│  PlanoController         PostgresPontoRepository                     │
│  ComprovanteController   GoogleMapsGateway (ACL)                     │
│  ExpediçãoPresenter      WhatsAppGateway (ACL) [Futuro]              │
│  PlanoPresenter          R2StorageGateway                            │
│        │                       │                                     │
└────────┼───────────────────────┼─────────────────────────────────────┘
         ▼                       ▼
┌──────────────────────────────────────────────────────────────────────┐
│  USE CASES (camada 2)                                                │
│                                                                      │
│  CriarExpedição          GerarPlanoDeEntrega                         │
│  AdicionarEntrega        ConfirmarPlano                              │
│  DefinirPontoNaEntrega   RegistrarComprovante                        │
│  RepriorizeEntrega       RegistrarOcorrência                         │
│  ListarEntregasPendentes BuscarCompradores                           │
│        │                                                             │
│  interfaces: ExpediçãoRepository, PontoRepository,                  │
│              CompradorRepository, PlanoRepository,                   │
│              ComprovanteRepository, MapsPort, StoragePort            │
│        │                                                             │
└────────┼─────────────────────────────────────────────────────────────┘
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  ENTITIES (camada 1)                                                 │
│                                                                      │
│  Expedição  Entrega  Ponto  PlanoDeEntrega  Comprovante  Comprador   │
│  LocalizaçãoPonto  PrazoDeEntrega  NívelUrgência  Volume  ...       │
│                                                                      │
│  ← nenhuma dependência externa                                       │
└──────────────────────────────────────────────────────────────────────┘
```

**Anti-Corruption Layers explícitas:**
- `GoogleMapsGateway` — traduz resposta da Maps API para `RotaCalculada` do domínio
- `WhatsAppGateway` — traduz `NotificaçãoEntrega` para formato da Meta/Twilio API
- `SupabaseAuthAdapter` — traduz sessão Supabase para `UsuárioAutenticado` do domínio

---

## 4. Inventário de Use Cases

### Módulo: Expedição

| Use Case | Ator | Descrição |
|---|---|---|
| `CriarExpedição` | Lojista | Cria nova expedição com zero ou mais entregas iniciais |
| `AdicionarEntregaAExpedição` | Lojista | Adiciona uma entrega (comprador + volumes) a uma expedição existente |
| `DefinirPontoNaEntrega` | Lojista | Define ou atualiza o ponto de uma entrega (quando comprador chega e informa) |
| `ListarExpediçõesPendentes` | Lojista | Retorna expedições prontas aguardando plano de entrega |
| `ListarEntregasAguardandoPonto` | Lojista | Retorna entregas sem ponto definido ainda |
| `AcompanharStatusExpedição` | Lojista | Status em tempo real de cada entrega (a caminho / entregue / ocorrência) |

### Módulo: Roteirização

| Use Case | Ator | Descrição |
|---|---|---|
| `GerarPlanoDeEntrega` | Sistema (IA) | Gera sugestão de plano ordenado por urgência (prazo) → setor → vaga |
| `ConfirmarPlanoDeEntrega` | Lojista | Lojista aprova o plano e atribui o entregador |
| `RepriorizeEntrega` | Lojista | Altera manualmente a posição de uma entrega no plano em andamento |
| `ObterProximaEntrega` | Entregador | Entregador solicita a próxima entrega a executar |

### Módulo: Comprovante & Ocorrência

| Use Case | Ator | Descrição |
|---|---|---|
| `RegistrarComprovante` | Entregador | Registra foto da mercadoria + foto da assinatura + GPS; marca entrega como `entregue` |
| `RegistrarOcorrência` | Entregador | Registra falha de entrega com motivo; marca entrega como `devolvida` |
| `ListarDevoluções` | Lojista | Retorna volumes devolvidos com motivo registrado |

### Módulo: Compradores & Pontos

| Use Case | Ator | Descrição |
|---|---|---|
| `BuscarCompradores` | Lojista | Busca por nome (retorna lista para desambiguação) |
| `SugerirPontoHabitual` | Sistema | Dado um `CompradorId`, retorna o ponto habitual pré-cadastrado |
| `ListarPontos` | Lojista/Entregador | Lista todos os pontos do polo com setor, vaga e horário de partida |
| `AtualizarPontoEmCampo` | Entregador | Corrige a localização de um ponto quando o comprador informa endereço diferente |

### Módulo: Identidade (Generic — via Supabase Auth)

| Use Case | Ator | Descrição |
|---|---|---|
| `AutenticarLojista` | Lojista | Login via e-mail/senha ou magic link |
| `AutenticarEntregador` | Entregador | Login via celular/senha |
| `CadastrarEntregador` | Lojista | Lojista adiciona um entregador ao seu time |

---

## 5. Estrutura de Pastas

```
src/
│
├── entities/                      # Camada 1 — zero dependências externas
│   ├── expedição.ts
│   ├── entrega.ts
│   ├── ponto.ts
│   ├── plano-de-entrega.ts
│   ├── comprovante.ts
│   ├── comprador.ts
│   └── value-objects/
│       ├── localizacao-ponto.ts
│       ├── prazo-de-entrega.ts
│       ├── nivel-urgencia.ts
│       ├── coordenadas-gps.ts
│       └── volume.ts
│
├── use-cases/                     # Camada 2 — depende só de Entities
│   ├── criar-expedicao/
│   │   ├── criar-expedicao.interactor.ts
│   │   ├── criar-expedicao.request.ts
│   │   └── criar-expedicao.response.ts
│   ├── gerar-plano-de-entrega/
│   ├── registrar-comprovante/
│   ├── registrar-ocorrencia/
│   ├── repriorize-entrega/
│   └── ... (um diretório por use case)
│
├── ports/                         # Interfaces definidas pelos Use Cases
│   ├── expedicao.repository.ts    # interface — implementada nos adapters
│   ├── ponto.repository.ts
│   ├── comprador.repository.ts
│   ├── comprovante.repository.ts
│   ├── plano.repository.ts
│   ├── maps.port.ts               # interface do Maps Gateway (ACL)
│   ├── storage.port.ts            # interface do Storage (fotos)
│   └── notificacao.port.ts        # interface de notificação [Futuro]
│
├── adapters/                      # Camada 3 — converte entre camadas
│   ├── controllers/
│   │   ├── expedicao.controller.ts
│   │   ├── plano.controller.ts
│   │   └── comprovante.controller.ts
│   ├── presenters/
│   │   ├── expedicao.presenter.ts
│   │   └── plano.presenter.ts
│   └── gateways/
│       ├── postgres-expedicao.repository.ts
│       ├── postgres-ponto.repository.ts
│       ├── postgres-comprador.repository.ts
│       ├── google-maps.gateway.ts    # ACL — Google Maps
│       ├── r2-storage.gateway.ts     # ACL — Cloudflare R2
│       └── whatsapp.gateway.ts       # ACL — Meta API [Futuro]
│
├── frameworks/                    # Camada 4 — glue code
│   ├── http/
│   │   └── server.ts              # Express/Hono — só roteia para controllers
│   ├── database/
│   │   ├── schema.ts              # Drizzle/Prisma schema
│   │   └── migrations/
│   └── auth/
│       └── supabase-auth.ts
│
└── main.ts                        # Composition root — conecta tudo
```

**Regra de ouro:** um arquivo em `entities/` ou `use-cases/` nunca faz `import` de algo em `adapters/` ou `frameworks/`. O linter/TypeScript path aliases enforça isso.

---

## 6. Interfaces de Repositório

Definidas na camada `ports/` — Use Cases dependem dessas interfaces, nunca das implementações.

```typescript
// ports/expedicao.repository.ts
export interface ExpedicaoRepository {
  save(expedição: Expedição): Promise<void>
  findById(id: ExpedicaoId): Promise<Expedição | null>
  findPendentesByLojista(lojistaId: LojistaId): Promise<Expedição[]>
}

// ports/ponto.repository.ts
export interface PontoRepository {
  findById(id: PontoId): Promise<Ponto | null>
  findBySetor(setorId: SetorId): Promise<Ponto[]>
  findAll(): Promise<Ponto[]>
  save(ponto: Ponto): Promise<void>
}

// ports/comprador.repository.ts
export interface CompradorRepository {
  findByNome(nome: string): Promise<Comprador[]>  // lista para desambiguação
  findById(id: CompradorId): Promise<Comprador | null>
  save(comprador: Comprador): Promise<void>
}

// ports/maps.port.ts
export interface MapsPort {
  calcularRota(
    origem: CoordenadasGPS,
    destinos: CoordenadasGPS[]
  ): Promise<RotaCalculada>
}

// ports/storage.port.ts
export interface StoragePort {
  armazenarFoto(arquivo: Buffer, tipo: 'mercadoria' | 'assinatura'): Promise<URL>
}
```

---

## 7. Decisões de Infraestrutura

### Banco de Dados

**Decisão:** PostgreSQL via Supabase.

| Critério | Justificativa |
|---|---|
| Relacional | Domínio tem relações claras (Expedição → Entrega → Ponto → Setor) |
| Supabase | Auth integrado, storage, realtime subscriptions — reduz infraestrutura para MVP |
| Realtime | `supabase.channel()` para atualizar status de entrega no painel sem polling |

### Armazenamento de Fotos (Comprovante)

**Decisão:** Supabase Storage (S3-compatible) no MVP; migrar para Cloudflare R2 se custo crescer.

Fotos de comprovante são dados críticos (segurança jurídica) — não podem ser perdidas.

### Mapas e Roteirização

**Decisão:** Google Maps API no MVP.

- `Directions API` para calcular rota entre loja e setor
- Referência de vaga (ex: "Setor Laranja - Vaga 33") guia entregador no campo
- **ACL obrigatória** — a resposta da Maps API nunca entra nos Use Cases diretamente
- Avaliar OpenStreetMap + OSRM pós-MVP para reduzir custo em volume

### Notificações

**Decisão:** Fora do MVP. Arquitetura já prevê o `NotificacaoPort` — quando implementado, não muda nenhuma linha de Use Case.

### Conectividade (PWA offline)

**Decisão:** Cache agressivo via Service Worker para operações críticas do entregador.

| Operação | Estratégia |
|---|---|
| Receber plano de entrega | Cache-first; atualiza ao conectar |
| Registrar comprovante | Queue local no Service Worker; sync quando online |
| Atualizar status de entrega | Online-required; alerta visual quando offline |

### Realtime

**Decisão:** Supabase Realtime (WebSocket) para o painel do lojista.

- Lojista vê status das entregas atualizando sem refresh
- Entregador envia comprovante → painel atualiza em segundos
- Sem necessidade de polling ou websocket manual no MVP

---

## 8. Stack Tecnológico Proposto

### Backend

| Componente | Escolha MVP | Alternativa |
|---|---|---|
| Linguagem | TypeScript (Node.js) | Python + FastAPI |
| Framework HTTP | Hono (edge-ready, leve) | Fastify |
| ORM | Drizzle ORM (type-safe, sem magic) | Prisma |
| Banco | PostgreSQL via Supabase | PostgreSQL standalone |
| Auth | Supabase Auth | Clerk |
| Storage | Supabase Storage | Cloudflare R2 |
| Deploy | Vercel (serverless) ou Railway | Fly.io |

### Frontend Web (Painel do Lojista)

| Componente | Escolha MVP | Alternativa |
|---|---|---|
| Framework | React + Vite | Next.js |
| State | Zustand + React Query | Jotai |
| Mapa | Google Maps JS API | Leaflet + OSM |
| Realtime | Supabase Realtime client | Socket.io |
| UI | shadcn/ui (Tailwind) | Mantine |

### App Mobile (Entregador — PWA)

| Componente | Escolha MVP | Futuro |
|---|---|---|
| Abordagem | PWA (mesmo codebase do web) | React Native |
| Câmera | Web API (`getUserMedia`) | Expo Camera |
| GPS | Web API (`Geolocation`) | Expo Location |
| Offline | Service Worker + IndexedDB | — |
| Push | Web Push API | FCM/APNs |

### IA (Sugestão de Plano de Entrega)

**Decisão:** Lógica determinística no MVP; Claude API (Anthropic) para wizard de voz no Lojista 2.

| Funcionalidade | Implementação |
|---|---|
| Ordenação do plano (prazo + setor + vaga) | Algoritmo determinístico — Entities layer |
| Desambiguação de comprador por nome | Busca fuzzy no `CompradorRepository` |
| Wizard de IA por voz `[Lojista 2]` | Claude API (streaming) |
| Futuro: repriorização automática | Claude API com contexto operacional |

---

## 9. Fluxos de Dados — Exemplo End-to-End

### Fluxo 1: Lojista cria expedição manual

```
Lojista → [Web UI]
  → HTTP POST /expedicoes
    → ExpedicaoController
      → constrói CriarExpedicaoRequest { lojistaId, entregas: [] }
        → CriarExpedicaoInteractor
          → cria Expedição (Entity) com status=rascunho
          → ExpedicaoRepository.save(expedição)
            → PostgresExpedicaoRepository → Supabase DB
          → retorna CriarExpedicaoResponse { expedicaoId }
        → ExpedicaoPresenter
      → HTTP 201 { expedicaoId }
  → [Web UI] navega para tela de expedição criada
```

### Fluxo 2: IA gera plano de entrega

```
Lojista → "Gerar Plano" → [Web UI]
  → HTTP POST /planos
    → PlanoController
      → constrói GerarPlanoRequest { expedicaoId }
        → GerarPlanoInteractor
          → busca Expedição com Entregas
          → para cada Entrega com ponto definido:
              → PontoRepository.findById(pontoId)
              → calcula PrazoDeEntrega (horarioPartida - tempoFila)
              → calcula NívelUrgência (critico / atencao / ok)
          → PlanoDeEntrega.criar(entregas) → ordena por urgência → setor → vaga
          → PlanoRepository.save(plano)
          → retorna PlanoResponse { itens ordenados, urgências }
        → PlanoPresenter → { plano com semáforo de urgência }
  → [Web UI] exibe plano para confirmação do lojista
```

### Fluxo 3: Entregador registra comprovante

```
Entregador → "Entregar" → [App PWA]
  → captura foto mercadoria (câmera) + foto assinatura + GPS
  → HTTP POST /comprovantes
    → ComprovanteController
      → constrói RegistrarComprovanteRequest { entregaId, fotos, coordenadas }
        → RegistrarComprovanteInteractor
          → StoragePort.armazenarFoto(fotoMercadoria) → urlFoto
          → StoragePort.armazenarFoto(fotoAssinatura) → urlAssinatura  ← obrigatória
          → cria Comprovante (Entity) — invariante: fotoAssinatura não nula
          → ComprovanteRepository.save(comprovante)
          → ExpedicaoRepository.marcarEntregaComoEntregue(entregaId)
          → emite evento EntregaConcluída (para notificação futura)
        → retorna RegistrarComprovanteResponse { comprovanteId }
  → [Web UI do Lojista] recebe update via Supabase Realtime
  → status da entrega atualiza para "entregue" no painel
```

---

## 10. Estratégia de Deploy

### MVP (2 lojistas, volume baixo)

```
GitHub
  └── GitHub Actions (CI)
        ├── testes unitários (Entities + Use Cases — sem infra)
        ├── testes de integração (Adapters contra Supabase test instance)
        └── deploy automático em merge para main
              ├── Backend (API) → Vercel Serverless Functions ou Railway
              └── Frontend → Vercel (CDN global)

Supabase (hosted):
  ├── PostgreSQL
  ├── Auth
  └── Storage
```

### Vantagens desta escolha para MVP:
- **Zero gestão de servidor** — foco no produto, não em operações
- **Custo próximo de zero** no nível de 2 lojistas (Supabase free, Vercel free)
- **Realtime incluso** no Supabase sem infra adicional
- **Escala automaticamente** quando o produto crescer

### Futuro (pós-validação)

| Componente | Evolução |
|---|---|
| Backend | Containerizar com Docker → migrar para Fly.io ou ECS |
| Banco | Manter Supabase ou migrar para RDS + réplicas de leitura |
| App mobile | Compilar React Native a partir do mesmo codebase lógico |
| Maps | Avaliar OpenStreetMap para reduzir custo de APIs em volume |
| Realtime | Adicionar Redis Pub/Sub se volume de conexões crescer |

---

## 11. Score do Modelo

Avaliação pelo Quick Diagnostic do Framework Clean Architecture:

| Critério | Status | Observação |
|---|---|---|
| Regras de negócio testáveis sem DB, servidor ou framework | ✅ | Entities e Use Cases são TypeScript puro — testam com `jest` sem nenhum mock de infra |
| Dependências apontam para dentro | ✅ | `entities/` não importa nada externo; `use-cases/` importa só `entities/` e `ports/` |
| Banco de dados trocável sem tocar regras de negócio | ✅ | `PostgresExpedicaoRepository` implementa `ExpedicaoRepository` — Use Cases não sabem que é Postgres |
| Use Cases independentes do mecanismo de entrega | ✅ | Use Cases recebem `Request` e retornam `Response` — sem `req`, `res` ou objetos Express |
| Framework confinado na camada externa | ✅ | Hono/Express vive em `frameworks/http/` — não é importado em `use-cases/` nem `entities/` |
| Grafo de componentes sem ciclos | ✅ | Fluxo unidirecional: frameworks → adapters → use-cases → entities |
| Main (composition root) conecta as dependências | ✅ | `main.ts` instancia todos os repositórios concretos e injeta nos Use Cases |

**Score: 7/7 → 10/10**

---

*Próximos passos: definição de stack final → setup do projeto → implementação por Use Case.*
