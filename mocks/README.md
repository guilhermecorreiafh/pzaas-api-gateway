# Mocks locais

Estes workflows permitem testar o Gateway localmente sem depender dos serviços das outras duplas.

## Cadastro e Identidade

Importe `wf_mock_cadastro_identidade.json` no n8n e ative o workflow. O mock mantém a `x-api-key` obrigatória (`turma2026`), mas nao exige nem utiliza `x-pedido-id` nas rotas de cadastro, login ou perfil.

| Operacao | Metodo e URL local | Resultado de sucesso |
|---|---|---|
| Cadastro | `POST /webhook/v1/mock/cadastro/cadastro` | `201`, token `token_mock_cliente_001` |
| Login | `POST /webhook/v1/mock/cadastro/login` | `200`, token `token_mock_cliente_001` |
| Perfil | `POST /webhook/v1/mock/cadastro/perfil` | `200`, perfil fixo do cliente mock |
| Health | `GET /webhook/v1/mock/cadastro/health` | `200`, `{"status":"tudo certo"}` |

Credenciais de login de sucesso:

```json
{"email":"cliente.mock@pzaas.test","senha":"senha-mock"}
```

Para usa-lo no Gateway, troque temporariamente as URLs dos nodes `req_cadastro`, `req_login` e `req_validar_perfil` para a mesma URL base local acima. Mantenha `x-api-key: turma2026`; nao envie `x-pedido-id` nesses tres caminhos.

Exemplo de verificacao direta:

```bash
curl -i -X POST "http://localhost:5678/webhook/v1/mock/cadastro/login" \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"email":"cliente.mock@pzaas.test","senha":"senha-mock"}'
```

## Entrega

`wf_mock_orquestrador_entrega.json` e um disparador manual que simula o Orquestrador notificando o Gateway. Ele agora chama `POST /webhook/v1/entrega` e envia `x-api-key: turma2026` e `x-pedido-id: 42`, conforme o contrato do Gateway.

Para testar com workflows em execucao manual, substitua `/webhook/` por `/webhook-test/` nas URLs dos mocks enquanto os respectivos workflows estiverem abertos no editor do n8n.
