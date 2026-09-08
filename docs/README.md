# PZaaS API Gateway Serviço 01

Documentação da API Gateway da pizzaria PZaaS, implementado em workflows do n8n. O serviço centraliza chamadas do cliente, valida os headers de integração e encaminha operações para Cadastro e Identidade, Catálogo, Pagamento e Orquestração de Pedido.

## Sumário

- [Visão geral](#visão-geral)
- [URLs e versionamento](#urls-e-versionamento)
- [Autenticação e correlação](#autenticação-e-correlação)
- [Endpoints](#endpoints)
- [Contratos detalhados](#contratos-detalhados)
- [Resiliência e integrações](#resiliência-e-integrações)
- [Códigos de resposta](#códigos-de-resposta)
- [Execução local](#execução-local)
- [Teste rápido](#teste-rápido)

## Visão geral

O Gateway expõe sete rotas. Cadastro e login são encaminhados ao Serviço 08. Cardápio consulta o Catálogo com cache em Redis. Pedido valida o cliente, consulta os preços do Catálogo, calcula o total e solicita o Pagamento. Status consulta duas versões do Orquestrador com fallback. Entrega recebe a notificação de que um pedido saiu para entrega.

```mermaid
flowchart LR
    C[Cliente] --> G[API Gateway]
    G --> I[Cadastro e Identidade]
    G --> R[(Redis)]
    G --> CA[Catálogo A]
    G --> CB[Catálogo B]
    G --> P[Pagamento]
    G --> OA[Orquestrador A]
    G --> OB[Orquestrador B]
```

| Componente | Uso |
|---|---|
| n8n | Publica webhooks, valida requisições e executa a orquestração. |
| HTTP e JSON | Formato de comunicação entre o cliente e os serviços. |
| Redis | Cache do cardápio por 300 segundos. |
| Postman ou curl | Testes manuais e validação do contrato. |

## URLs e versionamento

| Ambiente | URL base |
|---|---|
| Produção compartilhada | `https://pzaas.online/webhook` |
| n8n local | `http://localhost:5678/webhook` |
| Teste manual do n8n | `http://localhost:5678/webhook-test` |

As rotas públicas usam a versão `v1`. No n8n, `/webhook-test` funciona apenas enquanto o workflow está em modo de teste. Para integrações permanentes, ative o workflow e use `/webhook`.

## Autenticação e correlação

| Header | Regra |
|---|---|
| `Content-Type` | Use `application/json` nas requisições com body. |
| `x-api-key` | Obrigatório em todas as rotas, exceto no healthcheck. Valor de integração: `turma2026`. |
| `x-pedido-id` | Obrigatório em cardápio, status e entrega. Não é exigido em cadastro, login ou pedido. |
| `Authorization` | Obrigatório em cardápio, pedido e status no formato `Bearer <token>`. |

O Gateway valida o formato do Bearer e envia somente o token ao endpoint de perfil do Serviço 08. Uma resposta 401 vira `token_invalido`; outras falhas viram `autenticacao_indisponivel`.

## Endpoints

| Método | Rota | Finalidade | Bearer |
|---|---|---|---|
| GET | `/v1/health` | Informa a saúde do Gateway. | Não |
| POST | `/v1/gateway/cadastro` | Cria um usuário pelo Serviço 08. | Não |
| POST | `/v1/gateway/login` | Autentica um usuário pelo Serviço 08. | Não |
| GET | `/v1/cardapio` | Retorna o cardápio, com cache e fallback. | Sim |
| POST | `/v1/gateway/pedido` | Calcula o total e solicita o pagamento. | Sim |
| GET | `/v1/status` | Consulta o estado do pedido no Orquestrador. | Sim |
| POST | `/v1/entrega` | Recebe a notificação de saída para entrega. | Não |

## Contratos detalhados

### GET v1 health

Não exige headers nem body.

```bash
curl -i "https://pzaas.online/webhook/v1/health"
```

Resposta esperada `200 OK`:

```json
{"status":"ok","servico":"gateway","versao":"v1"}
```

### POST v1 gateway cadastro

Encaminha o cadastro ao Serviço 08 em `POST /cadastro_b/cadastro`.

```bash
curl -i -X POST "https://pzaas.online/webhook/v1/gateway/cadastro" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"nome":"Cliente Teste","email":"cliente@example.com","senha":"senha-segura"}'
```

Body:

```json
{"nome":"Cliente Teste","email":"cliente@example.com","senha":"senha-segura"}
```

Resposta de sucesso `201 Created`: o Gateway repassa o JSON do Serviço 08. Exemplo do contrato integrado:

```json
{"mensagem":"Usuario criado com sucesso","token":"uuid-gerado-pelo-banco"}
```

Erros próprios: `401` por API key e `503` se o Serviço 08 falhar.

### POST v1 gateway login

Encaminha as credenciais ao Serviço 08 em `POST /cadastro_b/login`.

```bash
curl -i -X POST "https://pzaas.online/webhook/v1/gateway/login" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"email":"cliente@example.com","senha":"senha-segura"}'
```

Resposta de sucesso `200 OK`:

```json
{"token":"uuid-do-banco"}
```

Erros: `401` por API key ou credenciais; `503` se o login estiver indisponível.

### GET v1 cardapio

Valida o token no Serviço 08 e procura o cardápio no Redis. Em cache miss, chama Catálogo A e, se houver erro, tenta Catálogo B. O resultado fica em cache por 300 segundos.

```bash
curl -i "https://pzaas.online/webhook/v1/cardapio" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: PED-003" \
  -H "Authorization: Bearer <token>"
```

Resposta `200 OK` repassada do Catálogo:

```json
{
  "pizzas": [{
    "id": 1,
    "nome": "Calabresa",
    "ingredientes": ["molho de tomate", "mussarela", "calabresa", "cebola"],
    "preco": 45.9,
    "disponivel": true
  }],
  "promocoes": [{"id":1,"descricao":"2 pizzas grandes por R$ 79,90","ativa":true}]
}
```

Erros: `400` sem `x-pedido-id`; `401` por API key, Bearer ausente ou token inválido; `503` se autenticação ou Catálogo falharem.

Chamadas reais executadas em `08/09/2026`:

```http
GET /webhook/v1/cardapio
x-api-key: invalid
x-pedido-id: DOC-TEST-001
Authorization: Bearer invalid

HTTP/1.1 401 Unauthorized
{"erro":"API key ausente ou invalida"}
```

```http
GET /webhook/v1/cardapio
x-api-key: turma2026
x-pedido-id: DOC-TEST-001
Authorization: Bearer token-invalido-documentacao

HTTP/1.1 401 Unauthorized
{"erro":"token_invalido"}
```

### POST v1 gateway pedido

Valida o token, consulta o Catálogo, calcula o total e chama o Pagamento. O preço enviado ao Pagamento é calculado pelo Gateway.

| Campo | Tipo | Obrigatório | Regra |
|---|---|---|---|
| `cliente_id` | string ou number | Sim | Identificador enviado ao Pagamento. |
| `token` | string | Sim | Use o mesmo token do Bearer. |
| `metodo_pagamento` | string | Sim | Método aceito pelo Pagamento. |
| `itens` | array | Sim | Lista de pizzas e quantidades. |
| `itens[].pizza` | string | Sim | Deve coincidir com `nome` no Catálogo. |
| `itens[].quantidade` | number | Sim | Multiplicador do preço. |
| `simular_erro` | boolean | Não | Apoio para o mock de Pagamento. |

```bash
curl -i -X POST "https://pzaas.online/webhook/v1/gateway/pedido" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "Authorization: Bearer <token>" \
  -d '{"cliente_id":19,"token":"<token>","metodo_pagamento":"PIX","itens":[{"pizza":"Calabresa","quantidade":2}],"simular_erro":false}'
```

Resposta aprovada usada pelo mock:

```json
{"status":"APROVADO","transacao_id":"trx_mock_0001","pedido_id":"ped_mock_0001","cliente":19,"valor":91.8}
```

Pizza sem preço no Catálogo retorna `422`:

```json
{"erro":"pedido_invalido","motivo":"pedido não atende às regras de validação"}
```

Uma recusa do Pagamento é repassada com o código recebido:

```json
{"status":"RECUSADO","codigo_erro":422,"motivo":"saldo_insuficiente"}
```

### GET v1 status

Valida o token e consulta o Orquestrador A. O identificador recebido em `x-pedido-id` é enviado ao Orquestrador A como query parameter `pedido_id` e ao Orquestrador B como header. Em falha de infraestrutura ou 5xx, o Gateway tenta o Orquestrador B. Um `404` do A é devolvido sem fallback.

```bash
curl -i -X GET "https://pzaas.online/webhook/v1/status" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: PED-004" \
  -H "Authorization: Bearer <token>"
```

Resposta `200 OK` repassada do Orquestrador. Exemplo do contrato fornecido:

```json
{"PedidoID":"PED-004","ClienteID":19,"SaborPizza":"Calabresa","status":"Pronto"}
```

Erros: `400` se `x-pedido-id` estiver ausente; `404` se não houver pedido; `503` se os Orquestradores falharem.

### POST v1 entrega

Recebe a notificação de saída para entrega. O body é ignorado; o identificador vem de `x-pedido-id`.

```bash
curl -i -X POST "https://pzaas.online/webhook/v1/entrega" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: PED-005" \
  -d '{}'
```

Resposta `200 OK`:

```json
{"recebido":true,"pedido_id":"PED-005","status":"saiu_para_entrega"}
```

## Resiliência e integrações

| Fluxo | Dependência | Comportamento |
|---|---|---|
| Cadastro e login | Serviço 08 em `/cadastro_b` | Falha externa vira `503`, exceto credenciais inválidas. |
| Validação de sessão | `POST /cadastro_b/perfil` | `401` vira `token_invalido`; demais erros viram `503`. |
| Cardápio | Catálogo A e B | Tenta B após falha do A; cache Redis por 300 s. |
| Pedido | Catálogo e Pagamento | Calcula o total com preços do Catálogo antes da cobrança. |
| Status | Orquestrador A e B | Tenta B em falha ou 5xx do A. |

Cache do cardápio:

- Chave: `pzaas:gateway:236161-237057:cardapio:v1`
- TTL: 300 segundos
- Conteúdo: resposta JSON completa do Catálogo
- Se o Redis falhar, o fluxo segue para o Catálogo A.

## Códigos de resposta

| HTTP | Situação |
|---|---|
| `200 OK` | Consulta, login, status, entrega ou healthcheck concluído. |
| `201 Created` | Cadastro concluído. |
| `400 Bad Request` | `x-pedido-id` ausente em cardápio, status ou entrega. |
| `401 Unauthorized` | API key, Bearer, token ou credenciais inválidas. |
| `404 Not Found` | Pedido não encontrado no status. |
| `422 Unprocessable Entity` | Pedido inválido ou recusa do Pagamento. |
| `503 Service Unavailable` | Dependência indisponível. |

| Campo `erro` | HTTP | Origem |
|---|---:|---|
| `API key ausente ou invalida` | 401 | Validação de `x-api-key`. |
| `x_pedido_id_ausente` | 400 | Header ausente. |
| `token_ausente` | 401 | Bearer ausente ou malformado. |
| `token_invalido` | 401 | Perfil respondeu 401. |
| `credenciais_invalidas` | 401 | Login recusado. |
| `cadastro_indisponivel` | 503 | Falha no cadastro. |
| `login_indisponivel` | 503 | Falha no login. |
| `autenticacao_indisponivel` | 503 | Falha na validação do perfil. |
| `Catálogo indisponível` | 503 | Falha nos dois caminhos de Catálogo. |
| `pedido_invalido` | 422 | Pizza sem preço no Catálogo. |
| `pedido_nao_encontrado` | 404 | Status sem resultado. |
| `orquestrador_indisponivel` | 503 | Falha nos caminhos de status. |

## Execução local

Pré-requisitos: Node.js com `npx`, Redis acessível para o cardápio e credenciais de integração configuradas no n8n.

```bash
npx n8n
```

Acesse `http://localhost:5678`, importe os arquivos de `workflows/` e configure as credenciais que não acompanham a exportação. Use **Execute workflow** com `/webhook-test` durante o desenvolvimento. Ative o workflow para usar `/webhook`.

```text
workflows/  workflows do Gateway, um JSON por rota
mocks/      serviços de apoio para testes locais
docs/       documentação e material da disciplina
```

- Nome de fluxo: `wf_gateway_<operação>`.
- Nodes e variáveis: minúsculas com `_`.
- Não edite os JSONs exportados à mão; altere no n8n e exporte novamente.
- Credenciais nunca devem ser commitadas.

## Teste rápido

1. Confirme localmente `GET /v1/health`.
2. Cadastre um usuário e guarde o token.
3. Faça login para confirmar as credenciais.
4. Consulte o cardápio com `x-pedido-id` e Bearer.
5. Crie um pedido usando nomes retornados pelo Catálogo.
6. Consulte o status com `x-pedido-id` no header.
7. Simule a notificação em `POST /v1/entrega`.

## Referências de integração

- [Cadastro e Identidade no GitHub](https://github.com/21lucasbarros/arquitetura-cloud)
- Workflows exportados em `../workflows/`
- Documentações das demais duplas usadas como referência de organização e exemplos
