# PZaaS — API Gateway (serviço 01)

Projeto da disciplina Arquitetura de Serviços em Nuvem — UNISANTA.
Dupla: Guilherme e (nome da dupla).

O Gateway é o ponto único de entrada da pizzaria. Ele recebe as chamadas do
cliente, gera o `x-pedido-id`, chama os outros serviços e devolve a resposta.

## Rodando local

Precisa de Node.js instalado.

```
npx n8n
```

Depois abre http://localhost:5678

## Estrutura

```
/workflows   os fluxos exportados do n8n (um arquivo por fluxo)
/mocks       fluxos falsos que simulam os serviços das outras duplas
/docs        contrato da API e coleção do Postman
```

## Convenções

- Fluxo: `wf_gateway_<endpoint>` (ex: `wf_gateway_health`)
- Gatilho: sempre começa com `gt_` (ex: `gt_health`) — exigência do professor
- Variáveis e nomes de node: sempre com `_`, tudo minúsculo
- Endpoints: sempre com `/v1` na frente

## Quem mexe no quê

Cada fluxo tem um dono. Só o dono edita.

| Fluxo | Dono |
|---|---|
| wf_gateway_health | |
| wf_gateway_cardapio | |
| wf_gateway_status | |
| wf_gateway_pedido | |

Se precisar mexer no fluxo do outro, avisa antes.

## Rotina de versionamento

O n8n de cada um roda na própria máquina. Quem manda é o GitHub.

Antes de começar a trabalhar:

1. `git pull`
2. Se algum arquivo em `/workflows` mudou, abre o n8n, menu `...` do fluxo →
   **Import from File** → escolhe o JSON. Agora sua máquina está igual à do outro.

Quando terminar e estiver funcionando:

3. No n8n, menu `...` do fluxo → **Download**
4. Salva o arquivo por cima do que já existe em `/workflows`
5. `git add . && git commit -m "descreve o que mudou" && git push`
6. Avisa a dupla que subiu

Para voltar atrás: `git checkout <commit> -- workflows/arquivo.json` e importa
o arquivo de novo no n8n.

## Coisas para não esquecer

- Credencial (senha do Redis, por exemplo) NÃO vai junto no JSON exportado.
  Cada um cria a credencial na própria máquina. Senha nenhuma entra no repo.
- Nunca editar JSON de fluxo na mão. Se der conflito no Git, escolhe um lado
  inteiro e refaz a alteração perdida pelo n8n.
- Nunca os dois editando o mesmo fluxo ao mesmo tempo. O n8n bloqueia, mas
  a regra vale para não perder trabalho.
