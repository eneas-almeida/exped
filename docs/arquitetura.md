# Arquitetura de Sistema — Exped (Clean Architecture)

Produzido com o framework Clean Architecture a partir do [discovery.md](discovery.md).

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
| **Express** | Roteamento HTTP; chama Controllers |
| **Prisma (SQL)** | ORM para PostgreSQL; confinado nesta camada |
| **Mongoose (NoSQL)** | Driver MongoDB para eventos e tracking; confinado nesta camada |
| **Google Maps SDK** | Consumido exclusivamente pelo Maps Gateway |
| **PostgreSQL** | Banco relacional; invisível para Use Cases |
| **MongoDB** | Banco de documentos para eventos e tracking; invisível para Use Cases |
| **JWT + bcrypt** | Autenticação; resultado chega aos Controllers como `UsuárioAutenticado` |
| **AWS ElastiCache (Redis)** | Cache de tokens, blacklist de refresh tokens, rate limiting |
| **AWS S3** | Armazenamento de fotos de comprovante e uploads futuros |
| **Datadog** | Logs, APM e alertas de infraestrutura |
| **Twilio / Meta API** | Envio de WhatsApp/SMS `[Futuro]` |

---

## 3. Mapa de Dependências

```
┌──────────────────────────────────────────────────────────────────────┐
│  FRAMEWORKS & DRIVERS (camada 4)                                     │
│                                                                      │
│  Express  PostgreSQL  MongoDB  Google Maps SDK  AWS S3  ElastiCache  │
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
│   ├── expedition.ts
│   ├── delivery.ts
│   ├── stop.ts
│   ├── delivery-plan.ts
│   ├── delivery-proof.ts
│   ├── buyer.ts
│   └── value-objects/
│       ├── stop-location.ts
│       ├── delivery-deadline.ts
│       ├── urgency-level.ts
│       ├── gps-coordinates.ts
│       └── volume.ts
│
├── use-cases/                     # Camada 2 — depende só de Entities
│   ├── create-expedition/
│   │   ├── create-expedition.interactor.ts
│   │   ├── create-expedition.request.ts
│   │   └── create-expedition.response.ts
│   ├── generate-delivery-plan/
│   ├── register-proof/
│   ├── register-occurrence/
│   ├── reprioritize-delivery/
│   └── ... (um diretório por use case)
│
├── ports/                         # Interfaces definidas pelos Use Cases
│   ├── expedition.repository.ts   # interface — implementada nos adapters
│   ├── stop.repository.ts
│   ├── buyer.repository.ts
│   ├── delivery-proof.repository.ts
│   ├── delivery-plan.repository.ts
│   ├── maps.port.ts               # interface do Maps Gateway (ACL)
│   ├── storage.port.ts            # interface do Storage (fotos)
│   └── notification.port.ts       # interface de notificação [Futuro]
│
├── adapters/                      # Camada 3 — converte entre camadas
│   ├── controllers/
│   │   ├── expedition.controller.ts
│   │   ├── delivery-plan.controller.ts
│   │   └── proof.controller.ts
│   ├── presenters/
│   │   ├── expedition.presenter.ts
│   │   └── delivery-plan.presenter.ts
│   └── gateways/
│       ├── postgres-expedition.repository.ts
│       ├── postgres-stop.repository.ts
│       ├── postgres-buyer.repository.ts
│       ├── google-maps.gateway.ts    # ACL — Google Maps
│       ├── s3-storage.gateway.ts     # ACL — AWS S3
│       └── whatsapp.gateway.ts       # ACL — Meta API [Futuro]
│
├── frameworks/                    # Camada 4 — glue code
│   ├── http/
│   │   └── server.ts              # Express — só roteia para controllers
│   ├── database/
│   │   ├── prisma/
│   │   │   ├── schema.prisma      # Schema PostgreSQL
│   │   │   └── migrations/
│   │   └── mongo-client.ts        # Mongoose — eventos e tracking
│   ├── cache/
│   │   └── redis-client.ts        # AWS ElastiCache (ioredis)
│   └── auth/
│       └── jwt-auth.ts            # JWT + bcrypt
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

### Domínio

**Decisão:** `exped.app`

O produto é acessado via `exped.app` (painel web do lojista e PWA do entregador no mesmo domínio). Subdomínios ou rotas específicas por perfil a definir na etapa de design.

### Multi-tenancy

**Decisão:** sistema multi-tenant — cada loja é um tenant isolado.

| Critério | Decisão |
|---|---|
| **Isolamento de dados** | Lojistas só acessam dados da própria loja; entregadores só acessam dados da loja que os cadastrou |
| **Estratégia** | Coluna `lojistaId` em todas as tabelas; middleware de autenticação injeta o `lojistaId` em cada query |
| **Escopo MVP** | Múltiplas lojas ativas no mesmo banco, sem particionamento físico |
| **Futuro** | Schema-per-tenant ou database-per-tenant se volume crescer |

### Hosting

**Decisão:** DigitalOcean Droplet (VPS único, Docker Compose).

| Critério | Decisão |
|---|---|
| **Servidor** | DigitalOcean Droplet — Ubuntu 24.04, 4 GB RAM, 2 vCPU (início) |
| **Containerização** | Docker + Docker Compose — backend, PostgreSQL, MongoDB, Nginx |
| **Reverse proxy** | Nginx — TLS via Let's Encrypt, roteamento `/api` → backend, `/` → frontend |
| **Orquestração** | Docker Compose — sem Kubernetes, Helm, Istio ou Argo no MVP |
| **Futuro** | Kubernetes + Helm quando escala horizontal for necessária |

### Banco de Dados

**Decisão:** PostgreSQL (dados relacionais) + MongoDB (eventos e tracking).

| Banco | Uso | Justificativa |
|---|---|---|
| **PostgreSQL 16** | Entidades do domínio (Expedição, Entrega, Ponto, Comprovante, Comprador) | Relações claras, transações ACID, integridade referencial |
| **MongoDB 7** | Eventos de tracking GPS, log de ocorrências, histórico operacional | Schema flexível para dados semi-estruturados; writes de alta frequência (GPS) |

Ambos rodam no mesmo Droplet via Docker Compose no MVP; migrar para serviços gerenciados (DO Managed Databases / MongoDB Atlas) quando volume justificar.

### Autenticação e Cache

**Decisão:** JWT + bcrypt para autenticação; AWS ElastiCache (Redis) para cache de tokens.

| Componente | Implementação |
|---|---|
| **Senha** | `bcrypt` (cost 12) |
| **Access token** | JWT assinado (1 hora de validade) |
| **Refresh token** | JWT de longa duração (7 dias); armazenado no ElastiCache para revogação imediata |
| **Blacklist** | Tokens revogados guardados no Redis até expiração natural |
| **Rate limiting** | Contadores por IP/usuário no Redis (`ioredis`) |
| **Futuro** | Adicionar autenticação social (Google, Apple) via Clerk ou Auth0 se necessário |

### Armazenamento de Arquivos

**Decisão:** AWS S3 para fotos de comprovante e uploads futuros.

Fotos de comprovante são dados críticos (segurança jurídica) — não podem ser perdidas. AWS S3 é o padrão consolidado para storage de objetos; custo previsível e integração madura com Node.js.

| Uso | Bucket | Configuração |
|---|---|---|
| Fotos de comprovante (mercadoria + assinatura) | `exped-comprovantes` | Privado; acesso via URL pré-assinada (15 min) |
| Uploads futuros (perfis, documentos) | `exped-uploads` | Privado; acesso por API |

### Observabilidade

**Decisão:** Datadog (logs, APM, alertas).

| Componente | Implementação |
|---|---|
| **Logs** | Winston + `datadog-winston` transport — logs estruturados em JSON |
| **APM** | `dd-trace` (auto-instrumentação de Express, Prisma, Mongoose, Redis) |
| **Métricas** | Datadog Agent como container no Compose — acessa `/var/run/docker.sock` para coletar CPU, memória, disco e métricas de todos os containers |
| **Alertas** | Monitors no Datadog para erros 5xx, latência p95, espaço em disco e falha de backup |

### Mapas e Roteirização

**Decisão:** Google Maps API no MVP.

- `Directions API` para calcular rota entre loja e setor
- **ACL obrigatória** — a resposta da Maps API nunca entra nos Use Cases diretamente
- Avaliar OpenStreetMap + OSRM pós-MVP para reduzir custo em volume

### Backup Diário

**Decisão:** backup automático diário para AWS S3.

| Banco | Estratégia | Ferramenta |
|---|---|---|
| **PostgreSQL** | Dump comprimido diário às 03h00 (UTC) | `pg_dump` via cron no Droplet |
| **MongoDB** | Dump comprimido diário às 03h15 (UTC) | `mongodump` via cron no Droplet |
| **Retenção** | 30 dias de backups diários | S3 Lifecycle Policy no bucket `exped-backups` |
| **Restauração** | Script documentado em `scripts/restore.sh`; testado mensalmente | `pg_restore` / `mongorestore` |

Script de backup: `scripts/backup.sh` — executado via cron, envia evento de sucesso/falha para Datadog.

### Realtime

**Decisão:** Socket.io para o painel do lojista.

- Lojista vê status das entregas atualizando sem refresh
- Entregador envia comprovante → painel atualiza em segundos
- Socket.io roda no mesmo processo Express no MVP

### Conectividade (PWA offline)

**Decisão:** Cache agressivo via Service Worker para operações críticas do entregador.

| Operação | Estratégia |
|---|---|
| Receber plano de entrega | Cache-first; atualiza ao conectar |
| Registrar comprovante | Queue local no Service Worker; sync quando online |
| Atualizar status de entrega | Online-required; alerta visual quando offline |

### Notificações

**Decisão:** Fora do MVP. Arquitetura já prevê o `NotificacaoPort` — quando implementado, não muda nenhuma linha de Use Case.

---

## 8. Stack Tecnológico

### Backend

| Componente | Escolha | Notas |
|---|---|---|
| Linguagem | TypeScript (Node.js 22 LTS) | — |
| Framework HTTP | **Express puro** | Leve, sem magia, encaixa perfeitamente no Clean Arch |
| ORM (SQL) | **Prisma** | Type-safe, migrations automáticas, schema declarativo |
| Driver NoSQL | **Mongoose** | ODM para MongoDB (eventos e tracking) |
| Banco relacional | PostgreSQL 16 | Dados do domínio; Docker no Droplet |
| Banco de documentos | MongoDB 7 | Eventos GPS, logs operacionais; Docker no Droplet |
| Cache | **AWS ElastiCache (Redis)** | Tokens JWT, blacklist, rate limiting (`ioredis`) |
| Autenticação | JWT + bcrypt | `jsonwebtoken` + `bcrypt`; sem dependência externa |
| Storage | **AWS S3** | Fotos de comprovante e uploads futuros |
| Observabilidade | **Datadog** | `dd-trace` + `datadog-winston` |
| Realtime | **Socket.io** | Painel do lojista; mesmo processo Express |
| Validação | **Zod** | Schema validation nos Controllers (Interface Adapters) |

### Frontend Web (Painel do Lojista)

| Componente | Escolha | Notas |
|---|---|---|
| Framework | **React 19 + Vite** | SPA; sem SSR necessário para painel autenticado |
| State | Zustand + TanStack Query | Estado global + server state com cache |
| UI | **shadcn/ui + Tailwind CSS** | Componentes copy-paste; total controle sobre o código |
| Mapa | Google Maps JS API | Visualização de rota e pontos de entrega |
| Realtime | Socket.io-client | Recebe updates de status do backend |
| Formulários | React Hook Form + Zod | Performance e validação declarativa |
| PWA | vite-plugin-pwa | Service Worker e manifest para o app do entregador |

### App Mobile (Entregador — PWA)

| Componente | Escolha | Futuro |
|---|---|---|
| Abordagem | PWA (mesmo codebase do web) | React Native |
| Câmera | Web API (`getUserMedia`) | Expo Camera |
| GPS | Web API (`Geolocation`) | Expo Location |
| Offline | Service Worker + IndexedDB | — |
| Push | Web Push API | FCM/APNs |

### IA (Sugestão de Plano de Entrega)

| Funcionalidade | Implementação |
|---|---|
| Ordenação do plano (prazo + setor + vaga) | Algoritmo determinístico — Entities layer |
| Desambiguação de comprador por nome | Busca fuzzy no `CompradorRepository` |
| Wizard de IA por voz `[Lojista 2]` | Claude API (streaming) |
| Futuro: repriorização automática | Claude API com contexto operacional |

### Infraestrutura e DevOps

| Componente | Escolha | Notas |
|---|---|---|
| Servidor | DigitalOcean Droplet | Ubuntu 24.04, 4 GB RAM, 2 vCPU (início) |
| Containerização | Docker + Docker Compose | Backend, PostgreSQL, MongoDB, Nginx |
| Reverse proxy | Nginx | TLS (Let's Encrypt), roteamento `/api` e `/` |
| CI/CD | **GitHub Actions** | Build → Test → Deploy via SSH |
| Cache / sessão | **AWS ElastiCache** | Redis gerenciado pela AWS |
| Storage | **AWS S3** | Fotos e uploads; backup diário |
| Observabilidade | **Datadog** | Logs, APM, métricas do Droplet, alertas |

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

**Domínio:** `exped.app`

```
GitHub
  └── GitHub Actions (CI/CD)
        ├── lint + type-check (TypeScript)
        ├── testes unitários (Entities + Use Cases — sem infra)
        ├── testes de integração (Adapters contra banco de teste local)
        └── deploy em merge para main
              └── SSH → DigitalOcean Droplet
                    └── docker compose pull && docker compose up -d

DigitalOcean Droplet — Docker Compose:
  ├── Nginx          (container — reverse proxy + TLS Let's Encrypt)
  ├── Express API    (container — node dist/main.js)
  ├── PostgreSQL 16  (container)
  ├── MongoDB 7      (container)
  └── Datadog Agent  (container — coleta logs/métricas via docker.sock; recebe traces do dd-trace na porta 8126)

AWS (serviços gerenciados):
  ├── ElastiCache (Redis) — cache de tokens e rate limiting
  └── S3 — fotos de comprovante + backups diários
```

### Pipeline GitHub Actions

```yaml
# .github/workflows/deploy.yml
jobs:
  test:
    steps:
      - npm ci
      - npm run lint
      - npm run type-check
      - npm run test:unit

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - Build imagem Docker → push GitHub Container Registry (ghcr.io)
      - SSH no Droplet → docker compose pull → docker compose up -d
      - Health check na API → evento de deploy no Datadog
```

### Sem orquestração complexa no MVP

| Ferramenta | Decisão | Futuro |
|---|---|---|
| Kubernetes | Não | Quando precisar de múltiplos pods ou zero-downtime rolling deploys |
| Helm | Não | Junto com K8s |
| Istio / service mesh | Não | Se múltiplos microserviços |
| Argo CD | Não | GitOps quando K8s entrar |

### Backup automático diário

```bash
# crontab no Droplet
# PostgreSQL — 03h00 UTC
0 3 * * * /opt/exped/scripts/backup-postgres.sh

# MongoDB — 03h15 UTC
15 3 * * * /opt/exped/scripts/backup-mongo.sh
```

Backups comprimidos enviados para `s3://exped-backups/YYYY-MM-DD/`. Retenção: 30 dias (S3 Lifecycle Policy). Alerta automático no Datadog em caso de falha.

### Futuro (pós-validação)

| Componente | Evolução |
|---|---|
| Banco | DO Managed Databases ou RDS (réplicas de leitura, failover automático) |
| App mobile | Compilar React Native a partir do mesmo codebase lógico |
| Maps | Avaliar OpenStreetMap para reduzir custo de APIs em volume |
| Filas | BullMQ + Redis para processamento assíncrono (notificações, relatórios) |
| K8s | Migrar quando escala horizontal ou múltiplos serviços forem necessários |

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
