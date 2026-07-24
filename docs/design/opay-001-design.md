# OPAY-001 — Design do fluxo confiável de solicitação de pagamento

- **Status:** Proposta para design review
- **Issue:** [#1](https://github.com/orionpay-labs/orionpay-platform/issues/1)
- **Responsável:** SDE II — Merchant Payments
- **Última atualização:** 2026-07-22

## 1. Resumo

A primeira versão da OrionPay receberá solicitações de pagamento em BRL, persistirá a intenção antes de acionar o provedor e permitirá consultar o estado do pagamento.

A solução será inicialmente um monólito modular em Spring Boot com PostgreSQL. A gravação do pagamento e do trabalho pendente para o provedor ocorrerá na mesma transação. Um worker processará esse trabalho de forma assíncrona, com entrega pelo menos uma vez. Chamadas repetidas ao provedor usarão uma chave estável e serão deduplicadas pelo provedor simulado.

A proposta não promete processamento "exactly once", pois não existe uma transação atômica entre o banco da OrionPay e um provedor externo. A garantia de negócio — não cobrar duas vezes — será obtida pela combinação de:

1. idempotência persistente na entrada da OrionPay;
2. restrições de unicidade no PostgreSQL;
3. transactional outbox;
4. chave idempotente estável em todas as tentativas junto ao provedor;
5. transições de estado condicionais e monotônicas;
6. consulta/reconciliação após resultados ambíguos.

## 2. Perguntas para Produto

| ID | Pergunta | Por que importa | Premissa enquanto estiver aberta |
|---|---|---|---|
| Q1 | Qual é o escopo da chave idempotente: global, por lojista ou por conta? | Define a restrição de unicidade e evita conflito entre lojistas. | A chave será única por lojista. |
| Q2 | Por quanto tempo a chave e o resultado devem ser retidos? | Reutilização após expiração pode gerar nova cobrança. | Retenção mínima de 30 dias; nenhuma reutilização nesse período. |
| Q3 | Quais campos definem que duas solicitações são equivalentes? | Necessário para detectar a mesma chave com payload diferente. | Lojista, referência da compra, valor, moeda e token do meio de pagamento. |
| Q4 | Um pagamento pode ser recusado definitivamente pelo provedor? | Define estados terminais e comportamento de retry. | Sim; recusa de negócio é terminal e não recebe retry. |
| Q5 | Qual latência é aceitável para o POST? | Determina se a API deve aguardar o provedor. | O POST confirma aceite durável, sem aguardar o provedor. |
| Q6 | Quais metas de volume, disponibilidade e latência devemos suportar? | Influencia índices, pool de conexões e capacidade do worker. | Projetar para 100 req/s, p95 do POST abaixo de 250 ms e disponibilidade mensal de 99,9%, como metas provisórias. |
| Q7 | Como o lojista é autenticado e identificado? | A identidade faz parte do escopo da idempotência e do controle de acesso. | Autenticação real fica fora desta iniciativa; um identificador de lojista de teste será obrigatório. |
| Q8 | O cliente precisa receber atualizações por webhook? | Altera o escopo da API e da entrega de eventos. | Não; nesta versão o cliente consulta o estado. |
| Q9 | Existe prazo máximo para um pagamento permanecer inconclusivo? | Define alertas, reconciliação e eventual intervenção operacional. | Após 15 minutos em estado não terminal, gerar alerta; não marcar falha automaticamente sem confirmação do provedor. |
| Q10 | Quais informações podem aparecer para operadores e lojistas nos erros? | Evita vazamento e define suporte operacional. | Mensagens externas serão estáveis e não conterão detalhes internos ou dados do meio de pagamento. |

## 3. Premissas adotadas

- Apenas BRL será aceito e valores serão representados em centavos inteiros; ponto flutuante não será usado.
- Cada requisição terá o header obrigatório `Idempotency-Key`, com 16 a 128 caracteres.
- A identidade do lojista será obtida de contexto autenticado no futuro. Localmente será simulada sem credenciais reais.
- O cliente enviará apenas um token fictício, por exemplo `tok_test_...`; PAN, CVV e dados reais de cartão serão rejeitados.
- Uma solicitação é considerada aceita somente depois do commit da transação no PostgreSQL.
- Se o banco estiver indisponível antes do commit, a API retorna erro e não afirma ter aceitado a solicitação.
- O provedor simulado oferece idempotência por chave e consulta por referência.
- Erros técnicos temporários podem ser repetidos; recusas de negócio não.
- O sistema usará relógio em UTC e identificadores UUID gerados pela aplicação.
- Webhooks, estorno, chargeback, conciliação financeira e produção real continuam fora do escopo.

## 4. Requisitos

### 4.1 Funcionais

- **RF-01:** aceitar uma solicitação de pagamento com valor positivo em BRL.
- **RF-02:** atribuir um `paymentId` único a cada operação aceita.
- **RF-03:** permitir consulta do pagamento por `paymentId`.
- **RF-04:** retornar o mesmo pagamento para a mesma combinação de lojista e chave idempotente.
- **RF-05:** retornar conflito determinístico quando a chave for repetida com conteúdo semanticamente diferente.
- **RF-06:** impedir duas cobranças diante de retries e concorrência.
- **RF-07:** persistir e recuperar trabalho pendente após reinicialização.
- **RF-08:** repetir operações temporariamente malsucedidas sem alterar sua identidade no provedor.
- **RF-09:** impedir transições inválidas de estado.
- **RF-10:** oferecer correlação entre requisição, pagamento, tentativa e operação no provedor.

### 4.2 Não funcionais

- **RNF-01 — Durabilidade:** nenhum pagamento aceito pode depender apenas de memória.
- **RNF-02 — Segurança:** dados reais de cartão não podem ser recebidos, persistidos ou registrados.
- **RNF-03 — Consistência:** idempotência deve funcionar entre instâncias concorrentes da aplicação.
- **RNF-04 — Observabilidade:** logs estruturados, métricas e identificadores de correlação devem permitir investigação ponta a ponta.
- **RNF-05 — Recuperação:** trabalho pendente deve voltar a ser processável automaticamente depois de reinício ou falha temporária.
- **RNF-06 — Reprodutibilidade:** aplicação e PostgreSQL devem iniciar localmente com instruções versionadas.
- **RNF-07 — Evolução:** domínio, persistência, API e integração com provedor devem ter limites claros dentro do monólito.
- **RNF-08 — Compatibilidade:** erros e estados externos devem possuir contratos documentados e estáveis.

As metas quantitativas de Q6 são provisórias e não constituem SLO aprovado até validação de Produto.

## 5. API proposta

### 5.1 Criar pagamento

`POST /v1/payments`

Headers:

- `Idempotency-Key: <chave-do-lojista>`
- `X-Merchant-Id: <lojista-de-teste>` apenas no ambiente local

Payload de exemplo:

```json
{
  "merchantReference": "order-123",
  "amount": 10990,
  "currency": "BRL",
  "paymentMethodToken": "tok_test_visa"
}
```

Resposta para uma nova solicitação aceita: `202 Accepted`.

Payload de Resposta `PaymentResponse`, valido para POST, replay e GET:

```json
{
   "paymentId": "9f27d8c1-63b4-4a0e-a7dc-2e914f6b8053",
   "merchantReference": "order-123",
   "amount": 10990,
   "currency": "BRL",
   "status": "PENDING",
   "createdAt": "2026-07-23T14:00:00Z",
   "updatedAt": "2026-07-23T14:00:00Z"
}
```


Resposta para replay equivalente: `200 OK`, header `Idempotent-Replayed: true` e representação atual do mesmo pagamento.

Resposta para a mesma chave com payload diferente: `409 Conflict`, código `IDEMPOTENCY_KEY_REUSED`.

O hash da requisição será calculado sobre uma representação canônica dos campos semanticamente relevantes. Headers transitórios, espaçamento JSON e correlation ID não participarão do hash.

### 5.2 Consultar pagamento

`GET /v1/payments/{paymentId}`

Retorna `200 OK` quando o pagamento pertence ao lojista solicitante e `404 Not Found` caso contrário. Não será revelado se um identificador existe para outro lojista.

### 5.3 Formato de erro

```json
{
  "code": "IDEMPOTENCY_KEY_REUSED",
  "message": "The idempotency key was already used with different payment data.",
  "correlationId": "6d203cc1-6b96-4c31-9a90-0cc09e6cff8b"
}
```

Códigos iniciais:

| HTTP | Código | Situação |
|---|---|---|
| 400 | `INVALID_REQUEST` | Campos ausentes, inválidos ou moeda não suportada. |
| 409 | `IDEMPOTENCY_KEY_REUSED` | Mesma chave com conteúdo diferente. |
| 404 | `PAYMENT_NOT_FOUND` | Pagamento ausente ou não acessível ao lojista. |
| 503 | `TEMPORARILY_UNAVAILABLE` | Não foi possível aceitar duravelmente a solicitação. |

## 6. Modelo de domínio e estados

Estados externos propostos:

- `PENDING`: aceito duravelmente e aguardando processamento;
- `PROCESSING`: existe tentativa junto ao provedor, mas o resultado ainda pode ser ambíguo;
- `SUCCEEDED`: confirmado pelo provedor; terminal;
- `FAILED`: recusado definitivamente ou falha terminal confirmada; terminal.

Transições permitidas:

```text
PENDING -> PROCESSING
PROCESSING -> PROCESSING
PROCESSING -> SUCCEEDED
PROCESSING -> FAILED
```

O pagamento só chega a FAILED quando o provedor devolve um resultado terminal comprovado:

```
PENDING
|
v
PROCESSING
|       \
v        v
SUCCEEDED  FAILED
```

Payload de exemplo para `FAILED`:

```json
{
  "paymentId": "9f27d8c1-63b4-4a0e-a7dc-2e914f6b8053",
  "status": "FAILED",
  "reasonCode": "PROVIDER_DECLINED",
  "message": "Payment was declined."
}
```

Uma atualização repetida para o mesmo estado é tratada como no-op idempotente quando os dados não conflitam. Estados terminais não voltam a estados não terminais. Atualizações usarão `version`/compare-and-set ou predicado sobre o estado atual, impedindo que respostas atrasadas sobrescrevam um resultado terminal.

Não haverá estado externo `UNKNOWN`. A ambiguidade operacional será representada por `PROCESSING`, acompanhada por dados internos de tentativas e próxima reconciliação.

## 7. Arquitetura proposta

### 7.1 Forma da aplicação

Monólito modular Spring Boot:

- **payment-api:** valida contrato, identidade do lojista e idempotency key;
- **payment-domain:** regras de negócio e máquina de estados;
- **payment-application:** casos de uso;
- **payment-persistence:** PostgreSQL, transações e outbox;
- **provider-adapter:** contrato e implementação do provedor simulado;
- **payment-worker:** despacho, retry e reconciliação;
- **observability:** correlação, métricas e logging seguro.

Esses limites são módulos lógicos, não microserviços. A separação permite testar e substituir o provedor sem introduzir custo operacional distribuído agora.

### 7.2 Persistência mínima

**payments**

- `id` UUID, chave primária;
- `merchant_id`;
- `idempotency_key`;
- `request_fingerprint`;
- `merchant_reference`;
- `amount_minor`;
- `currency`;
- referência/token fictício protegido;
- `status`;
- `version`;
- timestamps.

Restrição única: `(merchant_id, idempotency_key)`.

**provider_attempts**

- `id`;
- `payment_id`;
- `attempt_number`;
- `provider_idempotency_key` estável;
- resultado técnico sanitizado;
- timestamps e duração.

**outbox_events**

- `id`;
- `payment_id`;
- tipo;
- estado de processamento;
- número de tentativas;
- `available_at`;
- timestamps.

Índices serão derivados das consultas efetivas, incluindo itens de outbox disponíveis e pagamentos por lojista/id.

### 7.3 Fluxo de aceite e idempotência

1. A API valida apenas dados não sensíveis e calcula o fingerprint canônico.
2. Em uma transação curta, tenta inserir `payments` e o evento de outbox.
3. A restrição única no banco arbitra requisições concorrentes entre threads e instâncias.
4. Em conflito, a aplicação carrega o registro existente:
    - mesmo fingerprint: retorna o mesmo `paymentId`;
    - fingerprint diferente: retorna `409 IDEMPOTENCY_KEY_REUSED`.
5. A resposta só é emitida após o commit.
6. Se o commit ocorreu mas a resposta se perdeu, o retry encontra o mesmo registro.

A correção não dependerá de uma checagem "consultar e depois inserir", pois esse padrão possui race condition.

### 7.4 Despacho ao provedor

1. Worker: A ideia central é dividir o trabalho em duas transações curtas, com a chamada HTTP fora delas:
```
Transação 1: reivindicar trabalho
        ↓ commit
Sem transação: chamar provedor
        ↓
Transação 2: registrar resultado
```

Se mantivéssemos FOR UPDATE durante a chamada HTTP, uma lentidão do provedor manteria conexão e lock do PostgreSQL ocupados, causando contenção e possivelmente esgotando o pool.

`Worker reivindica o evento`

Em uma transação curta:

```sql
SELECT *
FROM outbox_events
WHERE available_at <= now()
  AND (
   (
      status IN ('PENDING', 'RETRY')
         AND (lease_until IS NULL OR lease_until <= now())
      )
      OR
   (
      status = 'IN_PROGRESS'
         AND lease_until <= now()
      )
   )
ORDER BY available_at, id
   LIMIT 1
FOR UPDATE SKIP LOCKED;
```

O worker atualiza o registro:
```
status        = IN_PROGRESS
lease_owner   = worker-17
lease_token   = UUID aleatório
lease_until   = agora + 60 segundos
attempt_count = attempt_count + 1
```

Também cria uma tentativa com estado STARTED e muda o pagamento de PENDING para PROCESSING, caso ainda esteja pendente.
Em seguida, faz commit. O FOR UPDATE termina aqui.
O lease_token identifica aquela aquisição específica. Mesmo o mesmo worker receberá outro token se adquirir novamente o evento.

`Chamada ao provedor sem transação`

Depois do commit:
`OrionPay -> HTTP -> provedor simulado`

A chamada utiliza:
`providerIdempotencyKey = paymentId`

O timeout HTTP deve ser menor que o lease. Por exemplo:

`Timeout HTTP: 10 segundos`
`Lease: 60 segundos`

Assim, normalmente o worker recebe sucesso, recusa ou timeout antes de perder o lease.


`Worker persiste o resultado`

Na segunda transação, a atualização da tentativa, a transição de estado do pagamento e a atualização do evento da outbox são realizadas atomicamente. 
A aplicação primeiro valida a posse do evento usando o lease_token e o prazo do lease. 
Se a posse for válida, as três alterações são persistidas no mesmo commit. Se qualquer atualização falhar, toda a transação sofre rollback.
Caso o lease tenha expirado ou o token não corresponda, nenhuma das três entidades é modificada.

Ao receber a resposta, abre uma nova transação e verifica se ainda é o proprietário:

Política (a), o worker só pode gravar enquanto o lease ainda estiver válido:

```sql
UPDATE outbox_events
SET status = 'DONE',
    processed_at = now(),
    lease_until = NULL
WHERE id = :eventId
  AND status = 'IN_PROGRESS'
  AND lease_token = :originalLeaseToken
  AND lease_until > now();
```

Se nenhuma linha for afetada, o lease já foi adquirido por outro worker. A resposta é considerada atrasada e esse worker não modifica o pagamento.

Sucesso confirmado

```
payment: PROCESSING -> SUCCEEDED
attempt: STARTED -> SUCCEEDED
outbox:  IN_PROGRESS -> DONE
```

Recusa definitiva

```
payment: PROCESSING -> FAILED
attempt: STARTED -> DECLINED
outbox:  IN_PROGRESS -> DONE
reasonCode: PROVIDER_DECLINED
```

Timeout ou erro temporário

```
payment: continua PROCESSING
attempt: STARTED -> UNKNOWN ou RETRYABLE_ERROR
outbox:  IN_PROGRESS -> RETRY
available_at: agora + backoff
lease_token: removido
lease_until: removido
```

O que acontece se o worker morrer?

| Momento do crash | Recuperação |
|---|---|
| Antes do primeiro commit | Nenhum lease foi adquirido; outro worker seleciona o evento. |
| Depois do commit, antes do HTTP | O lease expira e outro worker assume. |
| Durante o HTTP | O lease expira; a próxima tentativa usa a mesma chave idempotente. |
| Depois de o provedor cobrar, antes do segundo commit | Outro worker consulta ou repete com a mesma chave; o provedor devolve a operação original. |
| Worker antigo retorna depois da reaquisição | O `lease_token` não corresponde mais; a resposta atrasada não é aplicada. |

O SKIP LOCKED impede que dois workers adquiram simultaneamente o registro. O lease permite recuperação após o commit. O lease_token impede que um worker antigo aplique uma resposta depois de perder a posse.
Para o pagamento, use também uma atualização condicional:

```sql
UPDATE payments
SET status = 'SUCCEEDED',
    version = version + 1
WHERE id = :paymentId
  AND status = 'PROCESSING'
  AND version = :expectedVersion;
```
Estados terminais nunca são sobrescritos.
   

2. O pagamento passa de `PENDING` para `PROCESSING`.
3. Toda chamada usa uma chave derivada estável do `paymentId`; retries nunca geram uma nova chave de cobrança.
4. Em sucesso ou recusa confirmada, a transição terminal e o encerramento do evento são persistidos.
5. Em timeout, resposta inválida ou erro temporário:
    - o pagamento permanece `PROCESSING`;
    - a tentativa é registrada sem payload sensível;
    - o evento é reagendado com exponential backoff e jitter;
    - primeiro é consultado o resultado pela mesma referência ou repetida a operação idempotente.
6. Após reinício, itens não concluídos ou leases expirados voltam a ser elegíveis.

O provedor simulado manterá um registro durável por idempotency key: uma segunda chamada com a mesma chave retorna a operação original; a mesma chave com dados diferentes é rejeitada. Ele terá modos determinísticos de falha para timeout antes/depois do processamento, atraso, erro temporário e recusa.

### 7.5 Banco indisponível

- Antes do commit: retornar `503`; a operação não foi aceita.
- Depois do commit e antes da resposta: o cliente pode receber timeout, mas o retry recupera a mesma operação.
- Durante o worker: não chamar o provedor se não for possível obter/persistir o controle de processamento necessário. Se a falha ocorrer depois da chamada, tratar o resultado como ambíguo e reconciliar com a mesma chave.
- Backpressure impedirá retry agressivo durante indisponibilidade prolongada.

## 8. Segurança e dados sensíveis

- Rejeitar campos conhecidos de cartão, como PAN e CVV, no contrato.
- Aceitar somente tokens fictícios com formato allowlist em ambientes locais/testes.
- Não habilitar logging de bodies HTTP por padrão.
- Redigir headers de autorização, idempotency key e tokens em logs.
- Persistir apenas o mínimo necessário; o fingerprint será hash criptográfico, não cópia do payload.
- Usar queries parametrizadas e validação de tamanho para todos os campos.
- Não incluir exceções internas ou resposta bruta do provedor na API.
- Executar secret scanning e dependency scanning no CI quando configurados.
- Testes automatizados verificarão que PANs e tokens não aparecem em logs.
- Autorização por lojista será aplicada tanto no POST quanto no GET; o header local não é solução de produção.

## 9. Observabilidade e operação

Cada requisição terá `correlationId`; cada pagamento terá `paymentId`; cada tentativa terá `attemptId`. Logs serão estruturados e incluirão somente esses identificadores, estado anterior/novo, código de resultado e duração.

Métricas iniciais:

- contagem e latência de POST/GET por resultado;
- pagamentos aceitos e concluídos por estado;
- replays idempotentes e conflitos de fingerprint;
- tentativas do provedor por resultado;
- timeouts, retries e idade do item mais antigo da outbox;
- pagamentos em `PROCESSING` por faixa de idade;
- transições inválidas;
- erros de banco e saturação do pool.

Alertas propostos:

- item mais antigo da outbox acima de 5 minutos;
- pagamento em `PROCESSING` acima de 15 minutos;
- aumento sustentado de timeouts/erros do provedor;
- conflito idempotente anormalmente alto;
- falhas de banco ou worker sem progresso.

Um runbook deverá permitir buscar por `paymentId` ou `correlationId`, verificar histórico de tentativas, confirmar a operação no provedor e reprocessar com segurança. Reprocessamento manual sempre reutilizará a mesma chave; nunca criará outra cobrança.

## 10. Principais modos de falha e mitigação

| Falha | Efeito possível | Mitigação | Evidência esperada |
|---|---|---|---|
| Duas requisições simultâneas | Dois registros/cobranças | Unique constraint e tratamento do conflito | Teste concorrente com várias threads |
| Commit feito, resposta perdida | Cliente não sabe se aceitou | Retry encontra a mesma chave | Teste de falha após commit |
| Provedor cobra e ocorre timeout | Resultado ambíguo | Chave estável e consulta/retry idempotente | Teste timeout-after-process |
| Resposta atrasada chega após sucesso | Regressão de estado | Transição condicional e terminais monotônicos | Teste de reordenação |
| Aplicação reinicia | Trabalho em memória perdido | Outbox durável e recuperação de lease | Teste com reinício |
| Banco indisponível no POST | Falso aceite | Não responder sucesso antes do commit | Teste com PostgreSQL interrompido |
| Banco falha após chamada externa | Resultado não persistido | Reconciliação pela mesma chave | Teste de fault injection |
| Provedor retorna erro temporário | Pagamento parado | Retry com backoff/jitter e limite operacional | Teste de recuperação |
| Poison message | Loop infinito | Limite de tentativas, estado operacional e alerta; sem marcar cobrança como falha sem evidência | Teste de retries esgotados |
| Chave igual, payload diferente | Operação incorreta mascarada | Fingerprint canônico e 409 | Teste determinístico |
| Dados sensíveis em logs | Incidente de segurança | Allowlist, redaction e teste de captura de logs | Teste automatizado |
| Worker concorrente | Processamento duplicado | Lock/lease + idempotência no provedor | Teste com dois workers |

## 11. Estratégia de testes

### Testes unitários

- máquina de estados, incluindo todas as transições proibidas;
- canonicalização e fingerprint;
- classificação de erros temporários versus definitivos;
- cálculo de retry/backoff;
- validação de valor, moeda, chave e token fictício.

### Testes de persistência

Usar PostgreSQL real via Testcontainers, evitando depender de diferenças de comportamento de banco em memória:

- restrição única por lojista/chave;
- insert concorrente sincronizado por barreira;
- atomicidade entre pagamento e outbox;
- locking/lease entre dois workers;
- optimistic locking e transições condicionais.

### Testes de integração

- nova solicitação e consulta;
- replay sequencial equivalente;
- mesma chave com valor/token/referência diferente;
- dezenas de requisições concorrentes para a mesma chave resultando em um único pagamento;
- timeout antes do provedor processar;
- timeout depois do provedor processar;
- resposta atrasada e fora de ordem;
- erro temporário seguido de sucesso;
- recusa definitiva sem retry;
- reinício entre aceite e processamento;
- reinício durante o processamento;
- indisponibilidade temporária do banco;
- recuperação de evento/lease abandonado;
- ausência de informações sensíveis em banco, respostas e logs.

### Testes de Recuperação de Envento/lease Abandonado

- `Lease expirado sem reaquisição`: verificar que o worker antigo não consegue persistir o resultado depois de lease_until, mesmo que o lease_token ainda não tenha sido alterado.
- `Evento reaquirido por outro worker`: verificar que o novo worker recebe outro lease_token e que uma resposta atrasada do worker anterior não modifica tentativa, pagamento nem outbox.

Possíveis nomes:
```java
shouldRejectResultWhenLeaseHasExpired()
shouldRejectStaleWorkerAfterEventIsReacquired()
```

Esses testes comprovam especificamente a política (a) escolhida no documento.


### Testes de contrato

- OpenAPI validada contra endpoints;
- contrato do `PaymentProvider` executado para simulador e test doubles;
- códigos de erro e estados externos tratados como API pública.

### CI

O GitHub Actions executará build, formatação/lint, testes unitários, integração com PostgreSQL/Testcontainers, geração/validação OpenAPI e análise de dependências. Testes de concorrência serão escritos de forma determinística, com barreiras e timeouts explícitos, evitando `sleep` como mecanismo principal de sincronização.

## 12. Alternativas e trade-offs

### Chamada síncrona ao provedor dentro da transação do POST

**Rejeitada.** Parece simples, mas mantém transações de banco abertas durante I/O, aumenta contenção e ainda não cria atomicidade com o provedor. Um timeout continua ambíguo.

### Chamada síncrona depois do commit, sem outbox

**Rejeitada.** Uma queda entre o commit e a chamada perde o trabalho, violando recuperação após reinício.

### Transactional outbox no PostgreSQL

**Selecionada.** Garante que intenção e trabalho pendente sejam gravados atomicamente. Introduz worker, retry e manutenção da tabela, mas é proporcional ao escopo e não exige outra infraestrutura.

### Kafka ou fila gerenciada

**Adiada.** Pode melhorar desacoplamento e throughput, mas adiciona infraestrutura e ainda exige resolver dual write. O volume ainda não justifica.

### Lock distribuído em Redis

**Rejeitada.** Adiciona dependência e não substitui a restrição de unicidade durável. PostgreSQL já é a autoridade da operação.

### Isolamento SERIALIZABLE para todas as solicitações

**Não selecionado inicialmente.** Oferece garantias fortes, mas pode elevar aborts e retries. Unique constraint mais transações curtas resolve a disputa específica de idempotência.

### Simulador como microserviço de produção

**Rejeitado.** O simulador é ferramenta local/de teste atrás de uma interface. Não será implantado como serviço da OrionPay. Testes de integração poderão usar um processo HTTP isolado para reproduzir falhas de rede sem torná-lo componente de produção.

## 13. Backlog e ordem de execução

Estimativas são pontos relativos para uma pessoa e serão refinadas após as decisões abertas.

| Ordem | Cartão | Entrega verificável | Pontos | Dependência |
|---:|---|---|---:|---|
| 1 | OPAY-002 — Bootstrap do serviço | Spring Boot, build, PostgreSQL local e CI básico | 3 | Design aprovado |
| 2 | OPAY-003 — Contrato HTTP e OpenAPI | POST/GET e erros documentados, sem integração externa | 3 | OPAY-002 |
| 3 | OPAY-004 — Domínio e máquina de estados | Estados e invariantes com testes unitários | 3 | OPAY-002 |
| 4 | OPAY-005 — Persistência idempotente | Schema, migrações, fingerprint e concorrência | 5 | OPAY-003/004 |
| 5 | OPAY-006 — Transactional outbox | Gravação atômica e worker recuperável | 5 | OPAY-005 |
| 6 | OPAY-007 — Provedor simulado idempotente | Adapter, deduplicação e cenários de falha | 5 | OPAY-004 |
| 7 | OPAY-008 — Retry e reconciliação | Timeout, backoff, consulta e respostas atrasadas | 5 | OPAY-006/007 |
| 8 | OPAY-009 — Segurança de dados | Rejeição/redaction, testes e política de logs | 3 | OPAY-003 |
| 9 | OPAY-010 — Observabilidade | Correlação, métricas, dashboards locais e alertas propostos | 3 | OPAY-006/008 |
| 10 | OPAY-011 — Testes de falha e recuperação | Concorrência, banco, reinício e ambiguidades | 5 | OPAY-008/009 |
| 11 | OPAY-012 — Documentação operacional | Runbook, ADRs e instruções locais | 3 | OPAY-010/011 |

Total inicial: **43 pontos**. Uma faixa preliminar, não compromisso, é de **3 a 5 semanas** para uma pessoa, incluindo review e correções. O primeiro recorte vertical demonstrável termina em OPAY-008; segurança, observabilidade e testes de falha fazem parte da definição de pronto, não são opcionais.

## 14. Plano de rollout e rollback local

Mesmo sem produção real, a implementação deverá permitir evolução segura:

1. aplicar migrações compatíveis antes do código que as utiliza;
2. iniciar worker desabilitado por configuração;
3. validar criação, consulta e backlog de outbox;
4. habilitar processamento gradualmente;
5. observar idade da outbox, erros e pagamentos presos.

Em rollback, desabilitar o worker antes de reverter a aplicação. Dados e eventos não serão apagados. Migrações destrutivas não serão usadas nesta iniciativa. Após restaurar uma versão compatível, o worker continuará processando itens pendentes com as mesmas chaves.

## 15. Decisões e validações pendentes

- Respostas de Produto para Q1–Q10, especialmente retenção, identidade do lojista e SLOs.
- Formato exato e limite do `Idempotency-Key`.
- Campos finais que compõem o fingerprint.
- Política de retries, prazo de alerta e procedimento para pagamentos indefinidamente ambíguos.
- Tecnologia concreta do simulador HTTP nos testes.
- Definição de como tokens fictícios serão armazenados; preferencialmente referência opaca mínima.
- Aprovação dos estados externos e semântica HTTP de replay.
- Avaliação de privacidade/retenção antes de qualquer ambiente compartilhado.

## 16. Critérios de saída da design review

O design estará aprovado quando:

- Produto responder ou aceitar explicitamente as premissas das questões bloqueantes;
- estados, contrato HTTP e escopo de idempotência forem acordados;
- a estratégia para resultado ambíguo for aceita;
- riscos de segurança e operação tiverem responsáveis;
- backlog e primeiro recorte vertical estiverem priorizados;
- não restarem afirmações de "exactly once" sem mecanismo técnico correspondente.

Testes: não aplicável — documentação
