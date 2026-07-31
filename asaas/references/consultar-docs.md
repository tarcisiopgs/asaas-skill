# Consultar a documentação do Asaas

A doc é publicada em Markdown e indexada para agentes. Prefira estes caminhos a WebFetch genérico ou a scraping de HTML — o Markdown vem limpo e sem ruído de navegação.

## Sufixo `.md`

Qualquer página aceita `.md` no fim da URL e devolve o conteúdo em Markdown:

```
https://docs.asaas.com/docs/<slug>.md
https://docs.asaas.com/reference/<slug>.md
```

Exemplos reais:

```
https://docs.asaas.com/docs/split-de-pagamentos.md
https://docs.asaas.com/docs/sandbox.md
https://docs.asaas.com/docs/como-implementar-idempotencia-em-webhooks.md
```

Cada página vem com `updatedAt` no frontmatter — útil para saber se está olhando algo obsoleto.

## O índice: `llms.txt`

```
https://docs.asaas.com/llms.txt
```

Lista todas as páginas com uma linha de descrição cada, agrupadas em Guides, API Reference, Recipes, Pages e Changelog. Baixe este arquivo quando não souber o slug — é mais confiável do que adivinhar URLs, que retornam 404 silencioso.

Fluxo recomendado: pegar o `llms.txt`, localizar a página pelo texto da descrição, buscar aquela página com `.md`.

## MCP oficial

O Asaas mantém um servidor MCP público que expõe a especificação OpenAPI deles:

```
https://docs.asaas.com/mcp
```

Configuração típica:

```json
{
  "mcpServers": {
    "asaas-docs": { "url": "https://docs.asaas.com/mcp" }
  }
}
```

Aceita `?branch=<nome>` para fixar uma versão da documentação.

**É read-only sobre a documentação.** Ele responde "quais os parâmetros de tokenização de cartão?" — não cria cobrança, não consulta saldo, não toca na conta de ninguém. Para operar de fato, use a API REST com a chave.

## Seções do índice

| Seção | Conteúdo |
|---|---|
| Guides | Conceitos e passo a passo (autenticação, sandbox, split, webhooks) |
| API Reference | Endpoints, parâmetros, respostas |
| Recipes | Receitas prontas para casos comuns |
| Changelog | Mudanças de comportamento da API — vale checar antes de debugar algo que "funcionava antes" |
