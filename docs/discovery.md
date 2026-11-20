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

### Tipos de entrega

As expedições não são apenas intra-polo. Existem dois tipos:

| Tipo | Descrição |
|---|---|
| **Local (baia)** | Entrega no polo, na baia onde o comprador estacionou o veículo. O entregador percorre distâncias curtas dentro da área do polo. |
| **Regional** | Entrega para compradores que possuem loja física em outra cidade ou região. Os volumes são transportados pessoalmente pelos veículos próprios do lojista/entregador até o destino. |

### Escala nos dias de pico

Nos dias de maior movimento (ex: sexta-feira): **~100 pacotes** distribuídos em **~40 expedições** (média de 2–3 volumes por expedição). Isso define que o cadastro de cada expedição precisa ser muito rápido — se cada expedição levar 2 minutos, só o registro já consome mais de 1 hora.

### Como a baia é informada

A baia é comunicada pelo comprador ao lojista **geralmente no próprio dia**:
- O comprador estaciona o veículo, sai para fazer as compras e, quando chega na loja do lojista, informa o pedido e onde está estacionado (baia).
- Em alguns casos, compradores que fizeram pedido com antecedência chegam ao polo em um dia específico e **notificam o lojista que estão aguardando na baia** — só então o lojista organiza e despacha a entrega.

### Variações de fluxo por momento de compra

| Situação | Descrição |
|---|---|
| **Compra no mesmo dia** | Comprador chega, faz o pedido e informa a baia. Lojista cria a expedição com baia já definida. |
| **Compra com antecedência (dias antes)** | Comprador fez o pedido anteriormente. O volume já está embalado. No dia combinado, comprador chega ao polo e avisa o lojista em qual baia está. Lojista atualiza a expedição com a baia e despacha. |

Isso significa que uma expedição pode existir no sistema **sem baia definida** — aguardando o comprador chegar e informar. O sistema precisa representar esse estado.

---

## 2. Usuários

### Lojista
Responsável por cadastrar as expedições, organizar os volumes e acionar o entregador. É o usuário com maior fricção no processo atual — pouco tempo, muitos pedidos simultâneos. Pode não ter familiaridade com tecnologia.

### Entregador (pessoa que executa a entrega)
Pode ser o próprio dono da loja ou um funcionário registrado. Quem for executar uma entrega **precisa estar cadastrado no sistema** — não é possível registrar uma entrega sem um entregador identificado. Usa celular durante o trabalho para registrar o comprovante.

### Comprador (cliente do lojista)
Pode ser um comprador pessoa física ou dono de loja física em outra cidade/região. Chega ao polo de manhã (ou fez pedido dias antes), informa sua baia ao lojista e aguarda os volumes. **Não usa o sistema diretamente**, mas é o destinatário final das expedições — e pode receber uma notificação via WhatsApp quando a entrega for confirmada.

---

## 3. O Problema

O maior gargalo está no **cadastro das expedições**: o lojista não tem tempo para digitar os dados manualmente durante os picos de movimento.

A compra pode ter sido registrada em um sistema (soft) existente, ou pode estar anotada numa embalagem física. De qualquer forma, transformar essa informação em uma expedição cadastrada no sistema precisa ser rápido.

**Exemplo real de dado mínimo numa embalagem:** `Rafael – Alto Branco – 3 peças`. Com dados tão genéricos, o sistema precisa identificar qual "Rafael" é esse — pode haver múltiplos clientes com o mesmo nome — e associá-lo ao cliente correto cadastrado.

Outros problemas identificados:
- Falta de roteirização entre loja → baias nas expedições.
- Sem registro de comprovante de entrega.
- Sem visibilidade do status das entregas em tempo real.

---

## 4. Fontes de dados para uma expedição

Uma expedição pode ser criada a partir de três origens diferentes:

| Origem | Descrição | Status |
|---|---|---|
| **Voz (wizard de IA)** | Lojista fala ou digita numa conversa com a IA, que faz as perguntas, confirma o cliente e cria a expedição | **MVP — carro-chefe** |
| **Manual** | Formulário simples e eficiente para preenchimento direto | **MVP** |
| **Foto / upload de imagem** | OCR lê texto manuscrito ou impresso em qualquer superfície (papel, saco, etiqueta) | **Futuro (pós-MVP)** |
| **Integração com soft existente** | Puxa dados de compra já registrados no sistema do lojista | **Futuro (pós-MVP)** |

> **Decisão de MVP:** Foto/OCR e integração com soft são implementações futuras. O MVP foca em **voz (wizard IA) + manual**.

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

### Lojista — criação via wizard de IA (carro-chefe)

1. Lojista diz ou digita: **"nova expedição"**.
2. IA inicia um bate-papo e faz perguntas em sequência:
   - Quem é o cliente?
   - Qual o produto / quantos volumes?
   - Qual a baia?
   - Alguma observação?
3. IA confirma o cliente (desambiguação se necessário) e os dados.
4. Expedição é criada automaticamente.
5. IA sugere ou cria o **plano de rota** agrupando expedições pendentes.

### Lojista — criação manual

1. Lojista preenche formulário simples: cliente, volumes, baia.
2. Expedição é criada e entra na fila.

### Entregador (app mobile)

1. Recebe o plano de entrega no celular (pode ser o dono da loja ou funcionário cadastrado).
2. Faz a rota até as baias ou endereço de destino.
3. Ao entregar, registra o **comprovante de entrega**:
   - Foto da mercadoria no local.
   - **Foto da assinatura** de quem recebeu (campo obrigatório — segurança jurídica).
4. Sistema dispara automaticamente mensagem WhatsApp (ou SMS) ao comprador com:
   - Confirmação de entrega.
   - Foto da assinatura de quem recebeu.
5. Volta e recebe o próximo plano.

---

## 7. Glossário

| Termo | Definição |
|---|---|
| **Expedição** | Unidade de entrega — um ou mais volumes destinados a um comprador específico, a serem entregues na baia ou no endereço dele. Pode existir sem baia definida enquanto aguarda o comprador chegar. |
| **Baia** | Local fixo no polo onde o comprador estaciona seu veículo e aguarda os volumes. |
| **Expedição aguardando baia** | Estado de uma expedição criada com antecedência cujo comprador ainda não chegou ao polo — a baia será informada no dia da chegada. |
| **Plano de entrega** | Conjunto de expedições agrupadas para o entregador executar em uma saída. Criado/sugerido pela IA. |
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

### Wizard conversacional com IA — carro-chefe do produto
A principal interface de criação de expedições é uma conversa com a IA — por voz ou texto. O lojista não precisa navegar em formulários: apenas diz "nova expedição" e responde as perguntas da IA. Isso elimina o maior gargalo operacional sem exigir alfabetização digital.

### Desambiguação de clientes
Quando o nome informado for genérico (ex: "Rafael de Alto Branco"), a IA apresenta os clientes correspondentes cadastrados para confirmação, evitando entregas erradas.

### Criação automática do plano de rota
Após registrar expedições, a IA sugere ou cria automaticamente um plano de rota otimizado para o entregador, agrupando entregas por proximidade de baia.

### Notificação de entrega via WhatsApp / SMS
Quando o entregador registra o comprovante de entrega, o sistema dispara automaticamente uma notificação ao comprador com:
- Confirmação de que a mercadoria foi entregue.
- **Foto da assinatura** de quem recebeu (segurança para ambas as partes).

Canal de envio:
- **WhatsApp**: preferencial, se o comprador tiver WhatsApp cadastrado.
- **SMS**: fallback automático para compradores sem WhatsApp.
- **Telefone obrigatório**: campo obrigatório no cadastro do comprador. Sem ele, a notificação não pode ser disparada.

### Simplicidade radical
Pensado para pessoas sem familiaridade com tecnologia. Cada interação deve demandar o mínimo possível do lojista.

---

## 9. Restrições e Decisões de MVP

| Decisão | Detalhe |
|---|---|
| **Escopo geográfico** | Santa Cruz do Capibaribe – PE (teste inicial) |
| **Usuários de teste** | 2 lojistas |
| **Foco absoluto** | Reduzir tempo de cadastro de expedições |
| **Inputs do MVP** | Voz (wizard IA) + Manual |
| **Fora do MVP** | Foto/OCR, integração com soft existente, relatórios, múltiplos CDs, notificações ao comprador |
| **Concorrência** | Alta no setor geral; baixa no segmento de atacadistas de pequeno porte em polos regionais |

---

## 10. Em Aberto (perguntas não respondidas)

- Para pedidos com antecedência: o comprador recebe uma notificação — mas em qual direção? O sistema manda uma mensagem pro comprador perguntando a baia quando ele chegar? Ou é o comprador que avisa o lojista por fora (WhatsApp manual) e o lojista atualiza no sistema?
- Para entregas **regionais** (outras cidades), como funciona a logística? O entregador leva pessoalmente, entrega a uma transportadora, ou é outro modelo?
- O telefone do comprador precisa estar cadastrado previamente para receber notificação via WhatsApp? Como esse número é capturado — no momento do pedido, ou já existe no "soft"?
- Qual a proporção de entregas locais (baia) vs regionais numa manhã típica?
- Quem atribui a expedição ao entregador — o lojista manualmente ou a IA faz isso automaticamente?
- Quantas expedições um lojista cadastra numa manhã típica? (ordem de grandeza: 5, 20, 50?)
- O entregador é funcionário fixo da loja ou prestador avulso?
- O MVP será app mobile nativo ou PWA no browser?
- O sistema precisa funcionar com conectividade ruim (offline parcial)?
- O "soft" que o lojista usa hoje — há possibilidade de integração futura via API, ou é um sistema fechado?
