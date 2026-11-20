# Discovery — Exped.app

Registro vivo do processo de descoberta do produto. Atualizado à medida que novas informações surgem nas sessões de brainstorming.

---

## Índice

1. [Contexto Operacional](#1-contexto-operacional)
2. [Usuários](#2-usuários)
3. [O Problema](#3-o-problema)
4. [Fontes de dados para uma expedição](#4-fontes-de-dados-para-uma-expedição)
5. [Fluxo Atual (sem o produto)](#5-fluxo-atual-sem-o-produto)
6. [Fluxo Proposto (com o produto)](#6-fluxo-proposto-com-o-produto)
7. [Glossário](#7-glossário)
8. [Diferenciais](#8-diferenciais)
9. [Restrições e Decisões de MVP](#9-restrições-e-decisões-de-mvp)
10. [Em Aberto (perguntas não respondidas)](#10-em-aberto-perguntas-não-respondidas)
11. [Referência — App Carga MC (análise)](#11-referência--app-carga-mc-análise)
12. [Análise Crítica — Riscos, Gargalos e Diferenciação](#12-análise-crítica--riscos-gargalos-e-diferenciação)

---

## 1. Contexto Operacional

O MVP será testado em **Santa Cruz do Capibaribe – PE**, um dos maiores polos de moda do Brasil.

O funcionamento do polo é o seguinte:

- Compradores chegam cedo da manhã de van, ônibus, carro ou moto de diversas cidades do Nordeste.
- Eles estacionam em locais específicos chamados de **pontos** e saem para comprar produtos em diversas lojas.
- Cada comprador visita múltiplas lojas, fecha pedidos e precisa que os volumes sejam entregues no seu ponto antes de ir embora.
- O tempo disponível é curto — compradores não ficam esperando.
- O lojista precisa organizar os pedidos de cada comprador e despachar os volumes rapidamente até os pontos.
- Compras também podem ser feitas durante a semana ou dias antes — o lojista cria a embalagem e ela fica aguardando até o dia da entrega.
- Uma expedição pode ter **múltiplos volumes/pacotes** para o mesmo destinatário.

### Mapeamento geoespacial dos pontos

Os pontos do polo são estruturados em **setores** identificados por cor, cada um com coordenadas GPS. Dentro de cada setor, há **vagas numeradas** por tipo de veículo. A hierarquia de navegação do entregador é:

```
Polo → Setor (cor + GPS) → Vaga (número) → Tipo de veículo
```

Setores mapeados em Santa Cruz do Capibaribe:

| Setor | Cor | Lat | Lng |
|---|---|---|---|
| Laranja | 🟠 | -7.94935 | -36.22620 |
| Verde | 🟢 | -7.94900 | -36.22615 |
| Branco | ⚪ | -7.94885 | -36.22650 |
| Azul | 🔵 | -7.94930 | -36.22685 |
| Vermelho | 🔴 | -7.94920 | -36.22665 |
| Amarelo | 🟡 | -7.94898 | -36.22690 |
| Estacionamento Excursões | — | -7.94820 | -36.22750 |

Tipos de vaga por veículo:

| Tipo | Tempo médio de fila |
|---|---|
| **Vans** | ~15 min |
| **Pesado** (ônibus) | ~30 min |
| **Gerador** | ~20 min |

**Compradores recorrentes têm ponto fixo.** Muitos compradores sempre estacionam no mesmo ponto. Esses pontos podem ser pré-cadastrados e sugeridos automaticamente quando o comprador é selecionado — eliminando a necessidade de informar o ponto manualmente na maioria dos casos.

**Google Maps API** é utilizado para roteirização entre loja e setor (GPS). A referência de vaga (`Setor Laranja - Vaga 33`) guia o entregador ao ponto exato dentro do setor.

#### Modelo de dados do Ponto

O campo `setor_vaga_texto` do arquivo de brainstorming (`geo-espacial-clientes.json`) embute dados estruturados em texto livre — dificulta filtros e ordenação. O modelo recomendado para o sistema é separar os campos:

```json
{
  "nome": "Abdon Turismo",
  "tipo": "excursao_interna",
  "localizacao": {
    "setor": "laranja",
    "vaga": "33",
    "tipo_veiculo": "pesado"
  },
  "tempo_medio_fila_minutos": 30
}
```

Isso permite agrupar entregas por setor, ordenar por vaga dentro do setor e calcular tempo estimado de espera na fila.

### Excursões e horários de partida

Cada van, carro ou ônibus que chega ao polo é uma **excursão** e tem um **horário de partida** definido. Dentro de uma mesma excursão viajam **múltiplos compradores** de diferentes cidades.

Isso cria uma **restrição crítica de tempo**: a entrega de todos os volumes de um comprador precisa chegar ao ponto **antes do horário de partida da excursão** — caso contrário, o comprador vai embora sem a mercadoria.

| Conceito | Impacto operacional |
|---|---|
| Excursão com partida próxima | Entregadores precisam priorizar esses pontos primeiro |
| Múltiplos compradores na mesma excursão | Entregas para o mesmo ponto podem ser agrupadas numa única saída |
| Tempo médio de fila por tipo de veículo | Vans (~15min), Pesado (~30min) — deve ser somado ao tempo de deslocamento para calcular a margem real |

> O sistema deve exibir o **horário de partida** de cada excursão e alertar quando uma entrega corre risco de não chegar a tempo. O roteiro sugerido pela IA deve considerar os prazos — priorizando excursões com partida mais próxima.

O campo `horario_saida_padrao` já existe no arquivo de brainstorming (`geo-espacial-clientes.json`) mas está zerado. No sistema, esse horário deve ser cadastrado por excursão e servir como prazo-limite de entrega.

Modelo de dados atualizado:

```json
{
  "nome": "Abdon Turismo",
  "tipo": "excursao_interna",
  "localizacao": {
    "setor": "laranja",
    "vaga": "33",
    "tipo_veiculo": "pesado"
  },
  "horario_saida": "14:00:00",
  "tempo_medio_fila_minutos": 30
}
```

### Tipos de entrega

As expedições não são apenas intra-polo. Existem dois tipos:

| Tipo | Descrição |
|---|---|
| **Local (ponto)** `[MVP]` | Entrega no polo, no ponto onde o comprador estacionou o veículo. O entregador percorre distâncias curtas dentro da área do polo. |
| **Regional** `[Futuro]` | Entrega para compradores que possuem loja física em outra cidade ou região. Os volumes são transportados pessoalmente pelos veículos próprios do lojista/entregador até o destino. |

### Escala nos dias de pico

Nos dias de maior movimento (ex: sexta-feira): **~100 pacotes** distribuídos em **~50 expedições** (média de 2–3 volumes por expedição). Uma expedição é cadastrada de uma vez e pode conter **1 ou dezenas de entregas individuais** para diferentes destinatários. Isso define que o cadastro de cada expedição precisa ser muito rápido — se cada expedição levar 2 minutos, só o registro já consome mais de 1 hora e 40 minutos.

### Proporção local vs regional

**~90% das entregas são locais (ponto)** numa manhã típica. Entregas regionais são minoria e estão fora do escopo do MVP.

### Entregas regionais `[Futuro]`

O MVP foca exclusivamente em entregas locais no polo. Entregas regionais (outras cidades) serão endereçadas em versão futura. Quando implementadas, considera-se que o lojista possui veículo ou frota própria — o entregador leva pessoalmente os volumes ao destino.

### Como o ponto é informado

O ponto é comunicado pelo comprador ao lojista **geralmente no próprio dia**:
- O comprador estaciona o veículo, sai para fazer as compras e, quando chega na loja do lojista, informa o pedido e em qual ponto está estacionado.
- Para pedidos com antecedência: o comprador chega ao polo e **comunica ao lojista em qual ponto está** (geralmente por WhatsApp ou pessoalmente). A partir disso, o lojista identifica quais clientes estão naquele local e monta o roteiro de expedição para aquele ponto.

> O sistema deve permitir que o lojista veja facilmente **quais clientes já informaram o ponto** e quais ainda estão aguardando — para agrupar as entregas por destino e despachar de forma eficiente.

> **Canal de comunicação comprador → lojista:** O ponto é informado **fora do sistema** — via WhatsApp (contato comercial) ou ligação telefônica. O lojista recebe a informação e atualiza manualmente no sistema. O produto não gerencia essa comunicação.

### Variações de fluxo por momento de compra

| Situação | Descrição |
|---|---|
| **Compra no mesmo dia** | Comprador chega, faz o pedido e informa o ponto. Lojista cria a expedição com ponto já definido. |
| **Compra com antecedência (dias antes)** | Comprador fez o pedido anteriormente. O volume já está embalado. No dia combinado, comprador chega ao polo, avisa o lojista em qual ponto está. Lojista atualiza o ponto e o sistema mostra quais outros clientes também estão naquele ponto — facilitando o agrupamento do roteiro. |

Isso significa que uma expedição pode existir no sistema **sem ponto definido** — aguardando o comprador chegar e informar. O sistema precisa representar esse estado.

---

## 2. Usuários

### Lojista
Responsável por cadastrar as expedições, organizar os volumes e acionar o entregador. É o usuário com maior fricção no processo atual — pouco tempo, muitos pedidos simultâneos. Pode não ter familiaridade com tecnologia.

### Entregador (pessoa que executa a entrega) `[MVP]`
Pode ser o próprio dono da loja ou um prestador avulso que o lojista já conhece e aciona diretamente. No MVP, o lojista sabe quem vai entregar — o entregador é cadastrado no sistema e a atribuição acontece no momento de traçar o roteiro, não no cadastro da expedição. Quem for executar uma entrega precisa estar cadastrado no sistema. Usa celular durante o trabalho para registrar o comprovante.

### Entregador da região (autônomo avulso) `[Futuro]`
Existem entregadores que circulam pela região sem vínculo fixo com nenhuma loja. No futuro, esses entregadores poderão acessar o sistema para ver quais lojas estão precisando de entrega no momento e se deslocar até lá para se disponibilizar — modelo similar a um marketplace de entregas localizado. Essa funcionalidade abre o produto para um novo tipo de usuário e um novo modelo operacional.

### Comprador (cliente do lojista)
Pode ser um comprador pessoa física ou dono de loja física em outra cidade/região. Chega ao polo de manhã (ou fez pedido dias antes), informa seu ponto ao lojista e aguarda os volumes. **Não usa o sistema diretamente**, mas é o destinatário final das expedições — e pode receber uma notificação via WhatsApp quando a entrega for confirmada.

> **Telefone é opcional no cadastro.** Compradores sem telefone são casos raros e minoritários. O cadastro simplesmente fica sem número — o fluxo de entrega não para, a notificação simplesmente não é disparada.

---

## 3. O Problema

O maior gargalo está no **cadastro das expedições**: o lojista não tem tempo para digitar os dados manualmente durante os picos de movimento.

A compra pode ter sido registrada em um sistema (soft) existente, ou pode estar anotada numa embalagem física. De qualquer forma, transformar essa informação em uma expedição cadastrada no sistema precisa ser rápido.

**Exemplo real de dado mínimo numa embalagem:** `Rafael – Alto Branco – 3 peças`. Com dados tão genéricos, o sistema precisa identificar qual "Rafael" é esse — pode haver múltiplos clientes com o mesmo nome — e associá-lo ao cliente correto cadastrado.

Outros problemas identificados:
- Falta de roteirização entre loja → pontos nas expedições.
- **Risco de perder o horário de partida das excursões** — sem visibilidade dos prazos, o lojista não sabe quais entregas são urgentes e pode priorizar errado, resultando em mercadoria que não embarca.
- Sem registro de comprovante de entrega.
- **Sem rastreamento de devoluções** — volumes que voltam somem do controle; ninguém sabe o que retornou, por qual motivo ou o que fazer com eles.
- **Sem capacidade de repriorizar** — se um comprador avisa que vai embora em breve, o lojista não tem como reordenar o roteiro do entregador que já saiu.
- Sem visibilidade do status das entregas em tempo real.

---

## 4. Fontes de dados para uma expedição

Uma expedição pode ser criada a partir de três origens diferentes:

| Origem | Descrição | Status |
|---|---|---|
| **Manual** | Formulário simples e eficiente para preenchimento direto | **MVP** |
| **Voz (wizard de IA)** | Lojista fala ou digita numa conversa com a IA, que faz as perguntas, confirma o cliente e cria a expedição. Na UI, aparece como botão desabilitado sinalizando funcionalidade futura. | **Futuro** |
| **Foto / upload de imagem** | OCR lê texto manuscrito ou impresso em qualquer superfície (papel, saco, etiqueta) | **Futuro** |
| **Integração com soft existente** | Puxa dados de compra já registrados no sistema do lojista | **Futuro** |

> **Decisão de MVP:** O MVP foca exclusivamente no **cadastro manual**. Voz, foto/OCR e integração com soft são implementações futuras — o wizard de voz aparece desabilitado na interface como prévia da funcionalidade.

---

## 5. Fluxo Atual (sem o produto)

1. Comprador chega, fecha pedido na loja (ou fez pedido dias antes).
2. Lojista ou funcionário anota os dados manualmente (papel, WhatsApp, memória).
3. Volumes são embalados e acumulados na loja.
4. Lojista decide "na cabeça" quais volumes mandar de uma vez e chama o entregador.
5. Entregador pega os volumes, faz a rota entre loja e pontos, entrega um a um.
6. Entregador volta, pega mais volumes, repete o ciclo várias vezes no mesmo dia.
7. Nenhum comprovante formal de entrega. Sem rastreamento.

---

## 6. Fluxo Proposto (com o produto)

### Lojista — criação via wizard de IA `[Futuro]`

> Interface disponível na UI como botão desabilitado — sinaliza ao lojista que a funcionalidade está por vir.

1. Lojista diz ou digita: **"nova expedição"**.
2. IA inicia um bate-papo e faz perguntas em sequência:
   - Quem é o cliente?
   - Qual o produto / quantos volumes?
   - Qual o ponto?
   - Alguma observação?
3. IA confirma o cliente (desambiguação se necessário) e os dados.
4. Expedição é criada automaticamente.
5. IA sugere o **plano de rota** agrupando expedições pendentes por ponto/proximidade.
6. **Lojista escolhe o entregador** e confirma o plano.
   - _Evolução futura: IA atribui o entregador automaticamente com base em carga de trabalho e disponibilidade._

### Lojista — criação manual `[MVP]`

1. Lojista preenche formulário simples: cliente, volumes, ponto.
2. Expedição é criada e entra na fila.

### Entregador (app mobile) `[MVP]`

> **UX do app:** tela altamente intuitiva com botões grandes, pensada para uso em campo. O detalhamento de interface será feito na etapa de design. `[Design — fase futura]`

1. Recebe o plano de entrega no celular (pode ser o dono da loja ou funcionário cadastrado).
2. Faz a rota até os pontos.
3. Ao entregar, registra o **comprovante de entrega**: `[MVP]`
   - Foto da mercadoria no local.
   - **Foto da assinatura** de quem recebeu (campo obrigatório — segurança jurídica).
   - Entrega marcada como **concluída** no sistema.
4. Se a entrega não puder ser concluída, registra uma **ocorrência** com motivo: `[MVP]`
   - Comprador já foi embora (perdeu o prazo).
   - Comprador recusou a mercadoria.
   - Ponto não encontrado / endereço incorreto.
   - Outro motivo.
   - Volume retorna para a loja e entra no status **devolvido**.
5. Sistema dispara automaticamente mensagem WhatsApp (ou SMS) ao comprador. `[Futuro — pós-MVP]`
6. Volta e recebe o próximo plano.

### Lojista — gestão pós-despacho `[MVP]`

Após despachar o entregador, o lojista pode:

- **Acompanhar o status** de cada entrega (a caminho / entregue / devolvida / ocorrência).
- **Repriorizar entregas** — alterar manualmente a ordem do roteiro em andamento quando uma situação muda (ex: comprador avisa que está prestes a ir embora).
- **Receber devoluções** — volumes que retornam ficam visíveis no painel com o motivo registrado; lojista decide o que fazer (reagendar, contatar o comprador, etc.).

> **Repriorização automática pela IA** com base em alertas de prazo é evolução futura. `[Futuro]`

---

## 7. Glossário

| Termo | Definição |
|---|---|
| **Expedição** | Registro de saída criado pelo lojista — pode conter **1 ou dezenas de entregas individuais** para diferentes destinatários. É o objeto que o lojista cadastra no sistema e que agrupa tudo que precisa ser despachado. Pode existir sem entregador atribuído enquanto aguarda o momento do roteiro. |
| **Entrega** | Unidade individual dentro de uma expedição — um ou mais volumes destinados a **um comprador específico**, a serem entregues no ponto ou no endereço dele. Pode existir sem ponto definido enquanto aguarda o comprador chegar. |
| **Ponto** | Entidade registrada no sistema que representa onde um comprador estaciona no polo e aguarda os volumes. Pode ser uma excursão (van/ônibus com múltiplos compradores) ou um comprador independente com veículo próprio. Possui nome, tipo, localização (setor + vaga), horário de partida e tempo médio de fila. |
| **Localização do ponto** | Combinação de setor + vaga que define o endereço físico de um ponto no polo (ex: "Setor Laranja - Vaga 33"). Para entregas regionais, é o endereço completo do destino. |
| **Setor** | Área do polo identificada por cor (Laranja, Verde, Branco, Azul, Vermelho, Amarelo). Cada setor tem coordenadas GPS e agrupa vários pontos. É a unidade primária de agrupamento para a roteirização. |
| **Vaga** | Posição numerada dentro de um setor onde o veículo do comprador estaciona. Junto com o setor, forma a localização completa do ponto. |
| **Excursão** | Tipo de ponto — van ou ônibus que traz múltiplos compradores de outra cidade ao polo. Tem horário de partida definido que funciona como prazo-limite para todas as entregas daquele ponto. |
| **Comprador independente** | Tipo de ponto — comprador que chega com veículo próprio e estaciona numa vaga fixa. Pode ou não ter horário de saída definido. |
| **Horário de partida** | Hora prevista em que o ponto (excursão ou comprador independente) deixa o polo. Define o prazo máximo de entrega. Entregas que chegam após esse horário resultam em mercadoria não recebida. |
| **Prazo de entrega** | Derivado do horário de partida do ponto, descontado o tempo médio de fila. Usado pelo sistema para priorizar o roteiro. |
| **Entrega aguardando ponto** | Estado de uma entrega criada com antecedência cujo comprador ainda não chegou ao polo — o ponto será informado no dia da chegada. |
| **Plano de entrega / Roteiro** | Conjunto de entregas agrupadas para o entregador executar em uma saída. Criado/sugerido pela IA no momento do despacho. |
| **Wizard de IA** | Interface conversacional (voz ou texto) que conduz o lojista por perguntas para criar uma expedição rapidamente. |
| **Soft** | Sistema/software existente que o lojista já usa para registrar as compras dos clientes. |
| **Lojista** | Dono ou operador da loja que despacha os volumes. |
| **Entregador** | Pessoa cadastrada no sistema responsável por executar as entregas — pode ser o próprio dono da loja ou um funcionário. |
| **Comprovante de entrega** | Registro obrigatório feito pelo entregador ao concluir uma entrega: foto da mercadoria + foto da assinatura de quem recebeu. Marca a entrega como concluída no sistema. |
| **Ocorrência** | Registro de uma entrega que não pôde ser concluída — com motivo (comprador ausente, recusa, ponto incorreto etc.). O volume retorna à loja com status de devolvido. |
| **Devolução** | Volume que retornou à loja após uma tentativa de entrega malsucedida. Fica visível no painel do lojista com o motivo registrado para acompanhamento. |
| **Repriorização** | Alteração manual da ordem do roteiro em andamento pelo lojista, quando uma situação muda (ex: comprador avisou que vai sair em breve). |
| **Comprador** | Cliente do lojista; chega ao polo, compra e aguarda no ponto. Não usa o sistema diretamente. |
| **Polo** | Área geográfica do mercado atacadista (ex: Santa Cruz do Capibaribe). |
| **Volume** | Pacote físico individual (caixa, saco de nylon etc.). Uma expedição pode ter múltiplos volumes. |

---

## 8. Diferenciais

### Wizard conversacional com IA `[Futuro]`
Interface de criação de expedições via conversa com a IA — por voz ou texto. O lojista não precisa navegar em formulários: apenas diz "nova expedição" e responde as perguntas da IA. Elimina o maior gargalo operacional sem exigir alfabetização digital. No MVP, aparece como botão desabilitado na interface — sinaliza o diferencial futuro do produto sem bloquear o uso.

### Desambiguação de clientes `[MVP]`
Quando o nome informado for genérico (ex: "Rafael de Alto Branco"), a IA apresenta os clientes correspondentes cadastrados para confirmação, evitando entregas erradas.

### Sugestão de plano de rota pela IA `[MVP]`
Após registrar expedições, a IA sugere um plano de rota considerando dois critérios simultaneamente:
1. **Prazo de partida** — excursões com horário de saída mais próximo têm prioridade
2. **Proximidade geográfica** — agrupamento por setor e vaga para minimizar deslocamento

O lojista vê quais entregas estão urgentes (risco de perder a partida) e confirma o plano com o entregador.

> **Atribuição manual pelo lojista é MVP.** A IA decidindo automaticamente o entregador com base em carga de trabalho e disponibilidade é uma evolução futura. `[Futuro]`

### Notificação de entrega via WhatsApp / SMS `[Futuro — pós-MVP]`
Quando o entregador registra o comprovante de entrega, o sistema dispara automaticamente uma notificação ao comprador com:
- Confirmação de que a mercadoria foi entregue.
- **Foto da assinatura** de quem recebeu (segurança para ambas as partes).

Canal de envio:
- **WhatsApp**: preferencial, se o comprador tiver WhatsApp cadastrado.
- **SMS**: fallback automático para compradores sem WhatsApp.
- **Telefone opcional**: o cadastro do comprador aceita ausência de telefone. Sem número, a notificação simplesmente não é disparada — o fluxo de entrega segue normalmente.

### Marketplace de entregadores da região `[Futuro]`
Entregadores autônomos que circulam pelo polo poderão ver em tempo real quais lojas estão com expedições prontas aguardando entregador e se deslocar para se oferecer. Lojistas passam a ter acesso a um pool de entregadores disponíveis, sem precisar manter vínculo fixo com nenhum deles.

### Gestão de ocorrências e devoluções `[MVP]`
Quando uma entrega não pode ser concluída, o entregador registra o motivo no app. O volume entra automaticamente em status de devolvido e fica visível no painel do lojista. O lojista pode repriorizar manualmente entregas em andamento quando a situação mudar — como um comprador que avisou que está prestes a ir embora.

> Repriorização automática pela IA com base em alertas de prazo é evolução futura. `[Futuro]`

### Pontos pré-mapeados e sugestão automática de ponto `[MVP]`
Compradores recorrentes têm pontos fixos pré-cadastrados no sistema (setor + número da vaga). Quando o nome do comprador é informado na expedição, o sistema sugere automaticamente o ponto habitual — eliminando digitação manual na maioria dos casos.

### Simplicidade radical `[MVP]`
Pensado para pessoas sem familiaridade com tecnologia. Cada interação deve demandar o mínimo possível do lojista.

### Mapa gamificado com contagem regressiva `[Futuro — diferenciador estratégico]`
Experiência visual que transforma a operação de entregas em algo visivelmente urgente e controlado:
- **Mapa interativo do polo** (possivelmente 3D) mostrando a posição em tempo real do entregador.
- Cada ponto de entrega aparece numerado no mapa com indicador de urgência por cor (🔴 crítico / 🟡 atenção / 🟢 ok).
- **Contagem regressiva por entrega** — minutos restantes antes da excursão partir.
- Visualização de múltiplos entregadores simultâneos na tela do lojista.
- **Mecânicas de gamificação para entregadores:** sequência de entregas no prazo, pontuação por expedição, ranking interno.
- **Métricas ao vivo para o lojista:** % entregas no prazo, tempo médio de despacho, histórico de desempenho.

> Essa camada visual não tem equivalente em nenhum chatbot ou ferramenta genérica de IA. É o diferencial que torna o produto indispensável mesmo para quem conhece o ChatGPT.

---

## 9. Restrições e Decisões de MVP

| Decisão | MVP | Futuro |
|---|---|---|
| **Escopo geográfico** | Santa Cruz do Capibaribe – PE | Expansão para outros polos/regiões |
| **Usuários de teste** | 2 lojistas — com estratégia diferenciada (ver abaixo) | — |
| **Foco absoluto** | Reduzir tempo de cadastro de expedições | — |
| **Inputs de expedição** | Manual (lojista 1); Manual + Voz/IA (lojista 2) | Foto/OCR, integração com soft existente |
| **Atribuição do entregador** | Lojista atribui no momento do roteiro (já sabe quem vai entregar); expedição pode ficar "aguardando entregador" até o roteiro ser traçado | IA decide com base em carga e disponibilidade; marketplace de entregadores avulsos da região |
| **Plano de rota** | IA sugere agrupamento por setor → vaga; lojista confirma | IA otimiza e distribui automaticamente |
| **Roteirização** | Google Maps API para navegação até o setor; entregador usa referência de vaga para o ponto exato | Otimização automática entre múltiplos setores e vagas |
| **Notificação ao comprador** | Fora do MVP | WhatsApp / SMS automático ao confirmar entrega |
| **Relatórios operacionais** | Fora do MVP | Painel com métricas de desempenho |
| **Múltiplos CDs** | Fora do MVP | Suporte a múltiplos centros de distribuição |
| **Entregas regionais** | Fora do MVP — foco exclusivo em entregas locais no polo | Lojista usa veículo/frota própria; futuramente integração com transportadoras |
| **App mobile** | PWA (browser) | App nativo (iOS/Android) |
| **Conectividade** | Requer internet (tolerância a instabilidades pontuais) | Modo offline parcial com sincronização |
| **Integração com soft existente** | Fora do MVP (arquitetura deve ser aberta a integrações desde o início) | API aberta para integração com sistemas de gestão do lojista — alta prioridade pós-MVP |
| **Concorrência** | — | Alta no setor geral; baixa no segmento de atacadistas de pequeno porte em polos regionais |

### Abordagem de validação

O produto parte de hipóteses — o que achamos que vai funcionar. Quem vai lapidar essas hipóteses são os próprios lojistas usando o sistema gratuitamente. O objetivo nesta fase não é vender, mas **tornar o produto tão necessário que o uso manual se torne inaceitável** — criar dependência real onde hoje tudo é feito à mão com recursos pouco avançados.

> **Monetização está fora de escopo neste momento.** A prioridade é testar, validar e refinar até o produto ser genuinamente indispensável.

O modelo é iterativo e sequencial: cada lojista fornece feedbacks que alimentam ajustes antes do próximo. Com o tempo, isso constrói um banco de feedbacks e aprendizados que orienta a evolução do produto.

### Estratégia de teste — ciclo inicial

Os 2 lojistas iniciais são do mesmo contexto (mesmo polo), o que permite comparar resultados sem ruído de variáveis externas.

| | Lojista 1 | Lojista 2 |
|---|---|---|
| **Foco** | Validar o fluxo core | Validar o diferencial de voz |
| **Inputs disponíveis** | Somente manual | Manual + wizard de IA por voz |
| **Formato** | MVP básico | Protótipo funcional |
| **O que valida** | Se o cadastro manual já resolve o problema de velocidade; usabilidade real do formulário | Se a IA por voz reduz ainda mais a fricção; se o lojista adere ao modelo conversacional |
| **Risco principal** | Formulário ainda ser lento demais | IA errar o cliente ou a baia e gerar retrabalho |

Os feedbacks do lojista 1 informam os ajustes antes de lançar para o lojista 2. Os feedbacks do lojista 2 informam a entrada de um terceiro lojista — e assim por diante. O banco de feedbacks cresce a cada ciclo.

---

## 10. Em Aberto (perguntas não respondidas)

Todas as perguntas de discovery foram respondidas. Próximas questões surgirão nas etapas de design e desenvolvimento.

---

## 11. Referência — App Carga MC (análise)

Análise de 9 telas de um app similar desenvolvido para o contexto de atacadistas no Moda Center Santa Cruz. Usado como referência de UX e fluxo — não como concorrente direto.

### Fluxo completo identificado

| Tela | O que mostra | Insight para o Exped |
|---|---|---|
| **Login** | Seleção de papel: Lojista ou Entregador | Tagline "sem perder excursão" — confirma que o prazo é o problema central |
| **Nova entrega (1)** | Formulário: Cliente, Cidade, Excursão/Transportadora, Volumes, Tamanho (P/M/G), Horário limite, Observação | Cadastro por entrega individual, não por expedição inteira |
| **Nova entrega (2)** | Excursão selecionada → auto-preenche local ("Estacionamento A - Vaga 12 · Sai 17:00") e horário limite | **Maior insight de UX:** selecionar a excursão elimina 2 campos de uma vez |
| **Nova excursão (modal)** | Cadastro inline: Nome, Tipo, Setor/Vaga, Horário de saída | Criação sem sair do fluxo — evita navegação extra |
| **Lista de pacotes** | Cards com status visual (Aguardando carga / Entregue), nome do entregador atribuído | Lojista tem visão geral do status de cada entrega |
| **Montar carga (entregador)** | Lista com checkboxes — entregador seleciona o que vai levar nesta viagem | Suporta múltiplas viagens por turno; entregador controla a carga |
| **Seleção confirmada** | Checkbox marcado → card fica amarelo → "Iniciar Rota · N pacote(s)" fica ativo | Feedback visual imediato antes de confirmar |
| **Rota em andamento** | Mapa com posição atual → marcador numerado no destino + linha tracejada. "1159 m da vaga". Botão "Atualizar Local da Vaga" | App usa **OpenStreetMap** (não Google Maps — reduz custo). Botão de atualizar vaga é essencial: a localização pode mudar |
| **Entrega final** | Destaque da excursão + vaga. Botão "Entregar" com ícone de câmera → comprovante fotográfico | Comprovante capturado in-app, sem fluxo separado |

### O que o app de referência **não tem** (oportunidades para o Exped)

- Sem contagem regressiva de tempo até a partida.
- Sem priorização visual por urgência (semáforo de prazos).
- Sem gamificação ou métricas de desempenho.
- Sem mapa 3D ou visão do polo inteiro.
- Sem notificação automática ao comprador.
- Sem desambiguação de cliente por nome.
- Sem wizard de IA para criação de expedição.
- Sem gestão de devoluções.
- Sem histórico operacional estruturado.

### Decisões de UX validadas pela referência

1. **Excursão como âncora do formulário** — selecionar a excursão deve auto-preencher ponto e prazo. Reduz drasticamente a digitação.
2. **Criação inline de nova excursão** — sem navegação extra; modal direto na tela de cadastro.
3. **Checklist para o entregador** — o entregador decide o que carrega em cada viagem. Suporta múltiplas saídas no mesmo turno.
4. **"Atualizar Local da Vaga"** — o ponto informado pelo comprador pode estar errado ou mudar. O sistema deve permitir correção em campo.
5. **CTA único e grande para confirmar entrega** — "Cheguei na Vaga" e depois "Entregar" com câmera. Fluxo de 2 toques.
6. **Tamanho de volume como toggle (P/M/G)** — mais rápido que digitar; reduz erros.

---

## 12. Análise Crítica — Riscos, Gargalos e Diferenciação

### Gargalos operacionais identificados

| Gargalo | Descrição | Mitigação no Exped |
|---|---|---|
| **Velocidade de cadastro** | 50 expedições em pico → se cada uma levar 2 min = 1h40 só de registro. O MVP precisa fazer isso em <30 segundos por entrega | Excursão como âncora (auto-preenche ponto + prazo); ponto sugerido por histórico do comprador |
| **Nomes ambíguos** | "Rafael de Alto Branco" pode ser 3 clientes diferentes | Desambiguação com lista de sugestões ao digitar |
| **Ponto não informado ainda** | Comprador fez pedido com antecedência mas ainda não chegou | Estado "aguardando ponto" já documentado; sistema não bloqueia o cadastro |
| **Excursão com horário errado** | Se o horário de partida estiver incorreto no sistema, entregas perdem o prazo | "Atualizar Local da Vaga" + tela de edição rápida de horário; alerta quando prazo muda |
| **Múltiplas viagens do entregador** | O entregador não carrega tudo de uma vez; faz 3–5 viagens no mesmo turno | Checklist por viagem (como no Carga MC) — entregador seleciona o que leva agora |
| **Repriorização em campo** | Lojista descobre que comprador vai embora em 10 minutos com o entregador já fora | Repriorização manual pelo lojista; notificação push para o entregador (Futuro) |

### Riscos críticos

| Risco | Probabilidade | Impacto | Ação |
|---|---|---|---|
| **Formulário ainda lento demais** | Alta | Alto | Testar cronometrado com lojista 1 antes de lançar lojista 2 |
| **Baixa adoção por baixa literacia digital** | Média | Alto | UX radical: 1 ação por tela, botões grandes, zero jargão técnico |
| **Dados de ponto desatualizados** | Alta | Médio | Permitir que entregador atualize o ponto em campo; ciclo de revisão periódica |
| **Internet instável no polo em pico** | Alta | Alto | PWA com cache agressivo; operações críticas (confirmar entrega) devem funcionar com conectividade intermitente |
| **Concorrência do Carga MC** | Baixa-média | Médio | Diferenciação em urgência visual (countdown), gamificação e IA — o Carga MC não tem nenhum desses |
| **Lojista usa o produto mas não o entregador** | Média | Alto | O app do entregador precisa ser tão simples que a adoção não exija treinamento |
| **Churn pós-teste gratuito** | Desconhecida | Alto | Criar dependência de dados (histórico, pontos mapeados, comprovantes) que não existem fora do produto |

### Estratégia de diferenciação — por que o ChatGPT não resolve isso

O risco mais subestimado do produto é o lojista concluir que "uma IA genérica já faz isso". A resposta não é argumento — é experiência. O Exped precisa fazer coisas que um chatbot estruturalmente não consegue fazer:

| Capacidade | ChatGPT / IA genérica | Exped |
|---|---|---|
| Conhecer os pontos pré-mapeados do polo (GPS, vagas, horários) | ❌ Não tem | ✅ Base de dados local, construída e refinada com uso |
| Calcular urgência em tempo real (tempo até a partida da excursão) | ❌ Não tem relógio operacional | ✅ Motor de prazo em tempo real |
| Mostrar mapa com posição do entregador e destinos numerados | ❌ Não tem | ✅ Mapa ao vivo no app |
| Capturar foto + assinatura como comprovante jurídico | ❌ Não tem acesso à câmera | ✅ Comprovante em campo com GPS stamp |
| Notificar o comprador automaticamente ao entregar | ❌ Não tem integração com WhatsApp | ✅ Disparo automático pós-comprovante |
| Guardar histórico de quem entregou o quê, quando e com comprovante | ❌ Sem memória operacional | ✅ Rastreabilidade completa |
| Sugerir ponto habitual do comprador na hora do cadastro | ❌ Sem base de dados de compradores recorrentes | ✅ Aprende com uso; pré-sugere o ponto |
| Mapa gamificado com countdown + urgência visual | ❌ Impossível em chat | ✅ Diferencial de produto único |

**Mecanismo de dependência (moat):** quanto mais o lojista usa o Exped, mais valiosa fica a base de dados — histórico de compradores, pontos mapeados com precisão, horários reais de excursões, comprovantes arquivados. Essa camada de dados operacionais específicos do polo não existe em nenhum outro lugar e não pode ser replicada por um chatbot.

### Perguntas que ainda precisam de resposta

1. **Qual é o tempo médio atual** para o lojista registrar uma expedição manualmente (referência para medir o ganho real do produto)?
2. **O entregador usa o celular próprio?** Ou a loja fornece um aparelho? Isso impacta a escolha de PWA vs app nativo e o controle de câmera.
3. **Como o lojista vai saber que o entregador "chegou na vaga"?** O sistema precisa notificar o lojista quando o entregador confirmar chegada?
4. **O que acontece quando duas entregas são para o mesmo ponto mas com excursões diferentes?** Ex: vaga 33 tem duas vans de cidades diferentes partindo em horários diferentes.
