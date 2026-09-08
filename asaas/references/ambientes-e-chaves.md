# Ambientes, chaves de API e segurança

## URLs

| Ambiente | URL base |
|---|---|
| Produção | `https://api.asaas.com/v3` |
| Sandbox | `https://api-sandbox.asaas.com/v3` |

As chaves são específicas de cada ambiente — trocar a URL sem trocar a chave resulta em 401.

Se encontrar `https://sandbox.asaas.com/api/v3` em código existente, é o host antigo. Confirme na doc vigente antes de mantê-lo.

## Autenticação

Header próprio, não `Authorization`:

```http
access_token: $aact_prod_000...
```

## Prefixo revela o ambiente

| Prefixo | Ambiente |
|---|---|
| `$aact_prod_` | Produção |
| `$aact_hmlg_` | Sandbox (homologação) |

Isso permite uma verificação barata na inicialização: se o prefixo não bate com a URL base configurada, aborte com erro explícito. O caso perigoso é uma chave de produção apontando para o host de produção enquanto o desenvolvedor achava que estava em teste — cobranças reais, cartões reais, e-mails reais para clientes reais.

```ts
const isProd = baseUrl.includes("api.asaas.com") && !baseUrl.includes("sandbox");
const keyIsProd = apiKey.startsWith("$aact_prod_");

if (isProd !== keyIsProd) {
  throw new Error(
    `Ambiente inconsistente: URL ${isProd ? "produção" : "sandbox"} com chave ${keyIsProd ? "produção" : "sandbox"}`,
  );
}
```

## Preserve a chave conforme o runtime

A expansão de `$` depende de quem lê a configuração. Shell, carregador de `.env`, interpolação do Docker Compose e gerenciador de segredos podem aplicar regras diferentes. Não copie uma receita de escape entre eles.

- Identifique todas as camadas que carregam a variável e consulte a documentação de cada uma.
- Teste o caminho completo com um valor fictício contendo `$`; um escape incorreto também pode introduzir uma barra literal.
- Valide a presença e o prefixo esperado no processo, sem imprimir a chave real, partes dela ou headers de autenticação.
- Recarregue a configuração conforme o runtime; editar o arquivo não garante que o processo receba o novo valor.

Referências de configuração: [Node.js `.env`](https://nodejs.org/api/environment_variables.html#dotenv) e [interpolação no Docker Compose](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/). Use a documentação da ferramenta efetivamente utilizada pelo projeto.

## Gestão de chaves

- A chave é exibida **uma única vez** na criação. É irrecuperável — perdeu, gera outra.
- Até 10 chaves por conta, nomeáveis, com data de expiração opcional.
- Chaves podem ser desabilitadas e reabilitadas sem invalidar; exclusão é definitiva.
- Só usuários administradores geram chave, e só pela interface web (não pelo app).

## Higiene

- Nunca no código-fonte, nunca no front-end, nunca em log, nunca em ticket de suporte.
- Use gerenciador de segredos (Secrets Manager, Vault, 1Password) em vez de `.env` versionado.
- Se a chave de produção circulou em ambiente de desenvolvimento durante os testes finais, renove antes do go-live.
- Só HTTPS.

## Sandbox

O Sandbox simula clientes, cobranças, pagamentos, transferências e webhooks sem movimentar dinheiro real. Use durante todo o desenvolvimento e homologação.

Páginas úteis (com sufixo `.md`):

- `docs/sandbox` — visão geral e o que pode ou não ser testado
- `docs/como-configurar-sua-conta-no-sandbox` — configuração inicial
- `docs/como-adicionar-dinheiro-para-testes` — saldo fictício
- `docs/como-testar-funcionalidades` — simulação de pagamento e eventos

Antes de ir para produção, confirme: chave trocada, URL trocada, webhooks reapontados para o endpoint de produção, e o comportamento de erro testado (não só o caminho feliz).
