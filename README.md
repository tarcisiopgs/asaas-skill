# asaas-skills

Agent skill for integrating with [Asaas](https://www.asaas.com), a Brazilian payment gateway and digital account provider. Written in Portuguese, since Asaas' own documentation and audience are Brazilian.

Skill para agentes de IA sobre a API do Asaas — cobranças, split de pagamento, subcontas, webhooks e ambientes.

## Instalação

```bash
npx skills add tarcisiopgs/asaas-skills@asaas -g -y
```

O `-g` instala globalmente (nível de usuário), disponível em todos os agentes compatíveis — Claude Code, Cursor, Codex, Gemini CLI, entre outros.

Para instalar apenas no projeto atual, remova o `-g`.

## O que a skill cobre

| Tema | Conteúdo |
|---|---|
| **Convenções da API** | Valores em reais decimais (não centavos), header `access_token`, chave começando com `$` |
| **Ambientes** | URLs de produção e Sandbox, prefixo da chave por ambiente, verificação de consistência |
| **Cobranças** | `billingType`, Pix, boleto, cartão, tokenização, parcelamento, assinaturas |
| **Split de pagamento** | Base de cálculo no líquido, regra da própria carteira, bloqueio por divergência |
| **Subcontas** | `walletId`, restrição regulatória sobre a chave de API, o que não é herdado da conta raiz |
| **Webhooks** | Entrega at-least-once, idempotência por `id`, ordem de persistência e resposta |
| **Documentação** | Como consultar a doc oficial em Markdown, `llms.txt` e o MCP oficial |

O foco é o que a documentação não deixa óbvio à primeira leitura: as convenções que divergem de gateways internacionais e os erros que não estouram — passam, e aparecem depois na fatura do cliente ou no extrato do fim do mês.

## Estrutura

```
asaas/
├── SKILL.md
└── references/
    ├── ambientes-e-chaves.md
    ├── cobrancas.md
    ├── consultar-docs.md
    ├── split-e-subcontas.md
    └── webhooks.md
```

O `SKILL.md` carrega o essencial; as referências são lidas sob demanda conforme o tema da tarefa.

## MCP oficial do Asaas

O Asaas mantém um servidor MCP público com a especificação da API. É complementar a esta skill — ele responde sobre parâmetros e endpoints, a skill cobre decisões e armadilhas.

```bash
claude mcp add --scope user --transport http asaas-docs https://docs.asaas.com/mcp
```

## Fontes

Todo o conteúdo é derivado da documentação oficial em [docs.asaas.com](https://docs.asaas.com). Este é um projeto independente, sem vínculo com o Asaas.

Encontrou algo desatualizado ou incorreto? Abra uma issue ou um PR.

## Licença

MIT
