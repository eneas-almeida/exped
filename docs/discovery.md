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

---

## 1. Contexto Operacional

O MVP será testado em **Santa Cruz do Capibaribe – PE**, um dos maiores polos de moda do Brasil.

O funcionamento do polo é o seguinte:

- Compradores chegam cedo da manhã de van, ônibus, carro ou moto de diversas cidades do Nordeste.
- Eles estacionam em locais específicos chamados de **baias** e saem para comprar produtos em diversas lojas.
- Cada comprador visita múltiplas lojas, fecha pedidos e precisa que os volumes sejam entregues na sua baia antes de ir embora.
- O tempo disponível é curto — compradores não ficam esperando.
- O lojista precisa organizar os pedidos de cada comprador e despachar os volumes rapidamente até as baias.
- Compras também podem ser feitas durante a semana ou dias antes — o lojista cria a embalagem e ela fica aguardando até o dia da entrega.
- Uma expedição pode ter **múltiplos volumes/pacotes** para o mesmo destinatário.

### Mapeamento geoespacial das baias

As baias do polo são estruturadas em **setores** identificados por cor, cada um com coordenadas GPS. Dentro de cada setor, há **vagas numeradas** por tipo de veículo. A hierarquia de navegação do entregador é:

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

**Compradores recorrentes têm vaga fixa.** Muitos compradores sempre estacionam na mesma vaga. Esses pontos podem ser pré-cadastrados e sugeridos automaticamente quando o comprador é selecionado — eliminando a necessidade de informar a baia manualmente na maioria dos casos.

**Google Maps API** é utilizado para roteirização entre loja e setor (GPS). A referência de vaga (`Setor Laranja - Vaga 33`) guia o entregador ao ponto exato dentro do setor.

#### Modelo de dados sugerido para o ponto de entrega

O campo `setor_vaga_texto` do arquivo de brainstorming (`geo-espacial-clientes.json`) embute dados estruturados em texto livre — dificulta filtros e ordenação. O modelo recomendado para o sistema é separar os campos:

```json
{
  "nome": "Abdon Turismo",
  "tipo": "excursao_interna",
  "baia_padrao": {
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

Isso cria uma **restrição crítica de tempo**: a entrega de todos os volumes de um comprador precisa chegar à baia **antes do horário de partida da excursão** — caso contrário, o comprador vai embora sem a mercadoria.

| Conceito | Impacto operacional |
|---|---|
| Excursão com partida próxima | Entregadores precisam priorizar essas baias primeiro |
| Múltiplos compradores na mesma excursão | Entregas para a mesma baia podem ser agrupadas numa única saída |
| Tempo médio de fila por tipo de veículo | Vans (~15min), Pesado (~30min) — deve ser somado ao tempo de deslocamento para calcular a margem real |

> O sistema deve exibir o **horário de partida** de cada excursão e alertar quando uma entrega corre risco de não chegar a tempo. O roteiro sugerido pela IA deve considerar os prazos — priorizando excursões com partida mais próxima.

O campo `horario_saida_padrao` já existe no arquivo de brainstorming (`geo-espacial-clientes.json`) mas está zerado. No sistema, esse horário deve ser cadastrado por excursão e servir como prazo-limite de entrega.

Modelo de dados atualizado:

```json
{
  "nome": "Abdon Turismo",
  "tipo": "excursao_interna",
  "baia_padrao": {
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
| **Local (baia)** | Entrega no polo, na baia onde o comprador estacionou o veículo. O entregador percorre distâncias curtas dentro da área do polo. |
| **Regional** | Entrega para compradores que possuem loja física em outra cidade ou região. Os volumes são transportados pessoalmente pelos veículos próprios do lojista/entregador até o destino. |

### Escala nos dias de pico

Nos dias de maior movimento (ex: sexta-feira): **~100 pacotes** distribuídos em **~50 expedições** (média de 2–3 volumes por expedição). Uma expedição é cadastrada de uma vez e pode conter **1 ou dezenas de entregas individuais** para diferentes destinatários. Isso define que o cadastro de cada expedição precisa ser muito rápido — se cada expedição levar 2 minutos, só o registro já consome mais de 1 hora e 40 minutos.

### Proporção local vs regional

**~90% das entregas são locais (baia)** numa manhã típica. Entregas regionais são minoria e seguem outro ritmo operacional.

### Entregas regionais — logística no MVP

No MVP, considera-se que o lojista possui **veículo ou frota própria** para entregas regionais. O entregador (ou o próprio lojista) leva pessoalmente os volumes até o destino em outra cidade. Modelos alternativos (transportadoras, parceiros logísticos) são considerados evolução futura.

### Como a baia é informada

A baia é comunicada pelo comprador ao lojista **geralmente no próprio dia**:
- O comprador estaciona o veículo, sai para fazer as compras e, quando chega na loja do lojista, informa o pedido e onde está estacionado (baia).
- Para pedidos com antecedência: o comprador chega ao polo e **comunica ao lojista em qual baia está** (geralmente por WhatsApp ou pessoalmente). A partir disso, o lojista identifica quais clientes estão naquele local e monta o roteiro de expedição para aquela baia.

> O sistema deve permitir que o lojista veja facilmente **quais clientes já informaram a baia** e quais ainda estão aguardando — para agrupar as entregas por destino e despachar de forma eficiente.

> **Canal de comunicação comprador → lojista:** A baia é informada **fora do sistema** — via WhatsApp (contato comercial) ou ligação telefônica. O lojista recebe a informação e atualiza manualmente no sistema. O produto não gerencia essa comunicação.

### Variações de fluxo por momento de compra

| Situação | Descrição |
|---|---|
| **Compra no mesmo dia** | Comprador chega, faz o pedido e informa a baia. Lojista cria a expedição com baia já definida. |
| **Compra com antecedência (dias antes)** | Comprador fez o pedido anteriormente. O volume já está embalado. No dia combinado, comprador chega ao polo, avisa o lojista em qual baia está. Lojista atualiza a baia e o sistema mostra quais outros clientes também estão naquela baia — facilitando o agrupamento do roteiro. |

Isso significa que uma expedição pode existir no sistema **sem baia definida** — aguardando o comprador chegar e informar. O sistema precisa representar esse estado.

---

## 2. Usuários

### Lojista
Responsável por cadastrar as expedições, organizar os volumes e acionar o entregador. É o usuário com maior fricção no processo atual — pouco tempo, muitos pedidos simultâneos. Pode não ter familiaridade com tecnologia.

### Entregador (pessoa que executa a entrega) `[MVP]`
Pode ser o próprio dono da loja ou um prestador avulso que o lojista já conhece e aciona diretamente. No MVP, o lojista sabe quem vai entregar — o entregador é cadastrado no sistema e a atribuição acontece no momento de traçar o roteiro, não no cadastro da expedição. Quem for executar uma entrega precisa estar cadastrado no sistema. Usa celular durante o trabalho para registrar o comprovante.

### Entregador da região (autônomo avulso) `[Futuro]`
Existem entregadores que circulam pela região sem vínculo fixo com nenhuma loja. No futuro, esses entregadores poderão acessar o sistema para ver quais lojas estão precisando de entrega no momento e se deslocar até lá para se disponibilizar — modelo similar a um marketplace de entregas localizado. Essa funcionalidade abre o produto para um novo tipo de usuário e um novo modelo operacional.

### Comprador (cliente do lojista)
Pode ser um comprador pessoa física ou dono de loja física em outra cidade/região. Chega ao polo de manhã (ou fez pedido dias antes), informa sua baia ao lojista e aguarda os volumes. **Não usa o sistema diretamente**, mas é o destinatário final das expedições — e pode receber uma notificação via WhatsApp quando a entrega for confirmada.

> **Telefone é opcional no cadastro.** Compradores sem telefone são casos raros e minoritários. O cadastro simplesmente fica sem número — o fluxo de entrega não para, a notificação simplesmente não é disparada.

---

## 3. O Problema

O maior gargalo está no **cadastro das expedições**: o lojista não tem tempo para digitar os dados manualmente durante os picos de movimento.

A compra pode ter sido registrada em um sistema (soft) existente, ou pode estar anotada numa embalagem física. De qualquer forma, transformar essa informação em uma expedição cadastrada no sistema precisa ser rápido.

**Exemplo real de dado mínimo numa embalagem:** `Rafael – Alto Branco – 3 peças`. Com dados tão genéricos, o sistema precisa identificar qual "Rafael" é esse — pode haver múltiplos clientes com o mesmo nome — e associá-lo ao cliente correto cadastrado.

Outros problemas identificados:
- Falta de roteirização entre loja → baias nas expedições.
- **Risco de perder o horário de partida das excursões** — sem visibilidade dos prazos, o lojista não sabe quais entregas são urgentes e pode priorizar errado, resultando em mercadoria que não embarca.
- Sem registro de comprovante de entrega.
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
5. Entregador pega os volumes, faz a rota entre loja e baias, entrega um a um.
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
   - Qual a baia?
   - Alguma observação?
3. IA confirma o cliente (desambiguação se necessário) e os dados.
4. Expedição é criada automaticamente.
5. IA sugere o **plano de rota** agrupando expedições pendentes por baia/proximidade.
6. **Lojista escolhe o entregador** e confirma o plano.
   - _Evolução futura: IA atribui o entregador automaticamente com base em carga de trabalho e disponibilidade._

### Lojista — criação manual `[MVP]`

1. Lojista preenche formulário simples: cliente, volumes, baia.
2. Expedição é criada e entra na fila.

### Entregador (app mobile) `[MVP]`

> **UX do app:** tela altamente intuitiva com botões grandes, pensada para uso em campo. O detalhamento de interface será feito na etapa de design. `[Design — fase futura]`

1. Recebe o plano de entrega no celular (pode ser o dono da loja ou funcionário cadastrado).
2. Faz a rota até as baias ou endereço de destino.
3. Ao entregar, registra o **comprovante de entrega**:
   - Foto da mercadoria no local.
   - **Foto da assinatura** de quem recebeu (campo obrigatório — segurança jurídica).
4. Sistema dispara automaticamente mensagem WhatsApp (ou SMS) ao comprador com: `[Futuro — pós-MVP]`
   - Confirmação de entrega.
   - Foto da assinatura de quem recebeu.
5. Volta e recebe o próximo plano.

---

## 7. Glossário

| Termo | Definição |
|---|---|
| **Expedição** | Registro de saída criado pelo lojista — pode conter **1 ou dezenas de entregas individuais** para diferentes destinatários. É o objeto que o lojista cadastra no sistema e que agrupa tudo que precisa ser despachado. Pode existir sem entregador atribuído enquanto aguarda o momento do roteiro. |
| **Entrega** | Unidade individual dentro de uma expedição — um ou mais volumes destinados a **um comprador específico**, a serem entregues na baia ou no endereço dele. Pode existir sem baia definida enquanto aguarda o comprador chegar. |
| **Baia** | Local fixo no polo onde o comprador estaciona seu veículo e aguarda os volumes. Identificada por setor + vaga (ex: "Setor Laranja - Vaga 33"). |
| **Setor** | Área do polo identificada por cor (Laranja, Verde, Branco, Azul, Vermelho, Amarelo). Cada setor tem coordenadas GPS e agrupa várias vagas. É a unidade primária de agrupamento para a roteirização. |
| **Vaga** | Posição numerada dentro de um setor onde o veículo do comprador estaciona. Junto com o setor, forma o endereço completo da baia. |
| **Ponto de entrega** | Combinação de setor + vaga que identifica com precisão onde fazer a entrega local. Para entregas regionais, é o endereço completo do destino. |
| **Excursão** | Carro, Van ou ônibus que traz compradores de outra cidade ao polo. Pode ter múltiplos compradores a bordo — cada um com suas próprias compras. Tem um horário de partida definido que funciona como prazo-limite para todas as entregas daquela excursão. |
| **Horário de partida** | Hora prevista em que a excursão deixa o polo. Define o prazo máximo de entrega para todos os compradores que viajam nela. Entregas que chegam após esse horário resultam em mercadoria não recebida. |
| **Prazo de entrega** | Derivado do horário de partida da excursão do comprador, descontado o tempo médio de fila na baia. Usado pelo sistema para priorizar o roteiro. |
| **Entrega aguardando baia** | Estado de uma entrega criada com antecedência cujo comprador ainda não chegou ao polo — a baia será informada no dia da chegada. |
| **Plano de entrega / Roteiro** | Conjunto de entregas agrupadas para o entregador executar em uma saída. Criado/sugerido pela IA no momento do despacho. |
| **Wizard de IA** | Interface conversacional (voz ou texto) que conduz o lojista por perguntas para criar uma expedição rapidamente. |
| **Soft** | Sistema/software existente que o lojista já usa para registrar as compras dos clientes. |
| **Lojista** | Dono ou operador da loja que despacha os volumes. |
| **Entregador** | Pessoa cadastrada no sistema responsável por executar as entregas — pode ser o próprio dono da loja ou um funcionário. |
| **Comprovante de entrega** | Registro obrigatório feito pelo entregador: foto da mercadoria + foto da assinatura de quem recebeu. Enviado ao comprador via WhatsApp/SMS. |
| **Comprador** | Cliente do lojista; chega ao polo, compra e aguarda na baia. Não usa o sistema diretamente. |
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

### Pontos pré-mapeados e sugestão automática de baia `[MVP]`
Compradores recorrentes têm vagas fixas pré-cadastradas no sistema (setor + número da vaga). Quando o nome do comprador é informado na expedição, o sistema sugere automaticamente a baia habitual — eliminando digitação manual na maioria dos casos.

### Simplicidade radical `[MVP]`
Pensado para pessoas sem familiaridade com tecnologia. Cada interação deve demandar o mínimo possível do lojista.

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
| **Entregas regionais** | Lojista usa veículo ou frota própria | Integração com transportadoras/parceiros logísticos |
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
