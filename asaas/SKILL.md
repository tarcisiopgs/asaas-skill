---
name: asaas
description: Integração com a API do Asaas para cobranças, assinaturas, Pix Automático, split, subcontas, antecipação, Conta Escrow, webhooks e Sandbox. Use quando a tarefa mencionar explicitamente Asaas, endpoints do Asaas ou uma integração já identificada como Asaas, incluindo diagnóstico de autenticação e limites. Não ative apenas por termos genéricos como Pix, boleto, access_token ou marketplace.
---

# Asaas

A API do Asaas cobre cobrança (Pix, boleto, cartão), conta digital, split de pagamento e subcontas. Ela é REST, versionada em `/v3`, e as convenções dela divergem de gateways internacionais em pontos que causam bugs silenciosos — principalmente valores, autenticação e split.

Este arquivo cobre o que evita os erros mais caros. Os detalhes de cada área estão em `references/`, e a documentação oficial está sempre a uma requisição de distância (veja abaixo).

## Antes de responder: consulte a documentação real

A API muda e sua memória sobre ela envelhece. A documentação do Asaas é publicada em Markdown e é barata de consultar — não responda de memória sobre payloads, campos ou endpoints.

Toda página da doc tem uma versão `.md`: acrescente o sufixo à URL.

```
https://docs.asaas.com/docs/split-de-pagamentos  →  https://docs.asaas.com/docs/split-de-pagamentos.md
```

O índice completo (todas as páginas, com uma linha de descrição cada) está em `https://docs.asaas.com/llms.txt`. Baixe-o quando não souber qual página quer — é o mapa que evita adivinhar URLs.

Detalhes de navegação, referência de API e o MCP oficial: `references/consultar-docs.md`.

## As quatro divergências que quebram integrações

Se você está vindo de Stripe, Mercado Pago ou similar, estes quatro pontos são onde a intuição erra.

### 1. Valores são reais decimais, não centavos

`value: 100.50` é R$ 100,50. Gateways internacionais costumam usar o menor unidade monetária (centavos); o Asaas não.

Isso é perigoso porque não falha — passa. Um sistema que guarda centavos internamente e manda o inteiro cru cobra cem vezes o valor certo, e o erro só aparece na fatura do cliente. Se seu domínio armazena centavos, converta explicitamente na fronteira com o Asaas e deixe isso visível no código.

### 2. Autenticação é `access_token`, não `Bearer`

```http
access_token: $aact_prod_000...
```

Não é `Authorization: Bearer`. Header próprio, valor cru, sem prefixo de esquema.

### 3. Preserve o `$` inicial da chave

As chaves usam prefixos como `$aact_prod_` (produção) e `$aact_hmlg_` (Sandbox). A interpretação de `$` depende do shell, do carregador de `.env` e das camadas de configuração utilizadas. Não existe escape universal: consulte a documentação do runtime e valide a configuração com um valor fictício que contenha `$`, sem imprimir a chave real. Veja `references/ambientes-e-chaves.md`.

Confira também se a chave e a URL base pertencem ao mesmo ambiente antes de iniciar a integração.

### 4. Sandbox e produção têm hosts diferentes

| Ambiente | URL base |
|---|---|
| Produção | `https://api.asaas.com/v3` |
| Sandbox | `https://api-sandbox.asaas.com/v3` |

As chaves não são intercambiáveis entre ambientes. Se encontrar código apontando para `sandbox.asaas.com/api/v3`, é o host antigo — confirme na doc antes de manter.

## Split de pagamento

Split é como o Asaas resolve marketplace: a cobrança nasce em uma conta e parte do valor é creditada automaticamente em outras. Três regras concentram quase todos os bugs.

**A cobrança permanece na conta responsável pela venda ou serviço**, conforme a [documentação de split](https://docs.asaas.com/docs/split-de-pagamentos). Identifique essa conta pelo modelo da operação; não presuma que toda integração é um marketplace ou que a conta responsável será sempre uma subconta.

Quando a integração exigir autenticação como subconta, confirme a disponibilidade da chave antes de desenhar a arquitetura. Leia `references/split-e-subcontas.md`.

**Nunca inclua a própria carteira no split.** O que sobra depois dos splits já é creditado automaticamente a quem emitiu a cobrança. Mandar o próprio `walletId` faz a API retornar exceção — não é um no-op silencioso, é um erro que derruba a criação da cobrança.

**O percentual incide sobre o líquido, não sobre o bruto.** A base de cálculo é o `netValue`, que é o valor da cobrança menos a tarifa do Asaas. Uma cobrança de R$ 100 com tarifa de R$ 2 tem R$ 98 de base — um split de 50% transfere R$ 49, não R$ 50. Modelos de receita construídos sobre o bruto vão divergir da conta bancária todo mês.

Estados, bloqueio por divergência e limites de casas decimais: `references/split-e-subcontas.md`.

## Webhooks

O Asaas entrega eventos com garantia **at-least-once**: o mesmo evento pode chegar mais de uma vez, especialmente quando seu endpoint demora a responder. Isso não é falha, é o contrato — e integrações que assumem entrega única duplicam pedidos, e-mails e liberações de acesso.

Antes de persistir, valide o header `asaas-access-token` contra o token configurado para o webhook; esse segredo é distinto da chave da API. O padrão de deduplicação: cada evento traz um `id` estável entre reenvios. Persista esse `id` com restrição de unicidade, responda `200` assim que a persistência confirmar, e processe a regra de negócio depois, de forma assíncrona. Use inserção atômica com unicidade para reconhecer duplicatas e mantenha o identificador após concluir o processamento. O worker também precisa impedir efeitos duplicados em retries.

Responder `200` antes de persistir é a falha mais cara: o Asaas considera entregue e você perdeu o evento. Detalhes, exemplo de esquema e tratamento de fila pausada: `references/webhooks.md`.

## Depois que o dinheiro entra

Duas funcionalidades mexem em **quando** o valor fica disponível, e as duas afetam conciliação.

**Antecipação** adianta o recebimento cobrando uma taxa a mais. O detalhe que quebra integração de marketplace: se a cobrança tem split, a antecipação **muda a base de cálculo do repasse**, porque o líquido passa a descontar também a taxa de antecipação. Em split fixo a antecipação é recusada quando não cabe — falha visível. Em split percentual ela prossegue e o parceiro simplesmente recebe menos do que a conta feita sobre o bruto. Ninguém é avisado.

**Conta Escrow** faz o contrário: retém o que a subconta recebe até a liberação. Enquanto retido, o recebimento aconteceu mas o saldo não existe — e uma plataforma que trata "cobrança recebida" como "dinheiro disponível" diverge do extrato o tempo todo.

Detalhes, regras e exemplos: `references/antecipacao-e-garantia.md`.

## Três limites diferentes devolvem o mesmo 429

O Asaas limita por **rate limit** (por endpoint, com headers `RateLimit-*`), por **cota** (25.000 requisições por conta a cada 12h) e por **concorrência** (até 50 `GET` simultâneos). Os três respondem `429`, então o código sozinho não diz qual estourou.

Isso importa porque a reação é oposta: retry com backoff resolve o rate limit, mas em cota só queima mais do orçamento de 12h, e em concorrência adiciona mais uma requisição à fila que já transbordou. Leia os headers antes de reagir. Detalhes em `references/limites-e-erros.md`.

## Comparação com outros provedores

Quando a tarefa pedir uma comparação, parta dos requisitos informados: meios de pagamento, parcelamento, recorrência, repasses, disponibilidade regional e custos. Não presuma que Pix, marketplace ou faturamento recorrente são necessários para todo produto. Confirme recursos e restrições na documentação vigente de cada provedor.

## Referências

| Arquivo | Quando ler |
|---|---|
| `references/consultar-docs.md` | Navegar a doc oficial, referência de API, MCP |
| `references/cobrancas.md` | Criar cobrança, Pix, boleto, cartão, parcelamento, assinatura |
| `references/split-e-subcontas.md` | Marketplace, subcontas, `walletId`, estados de split |
| `references/webhooks.md` | Receber eventos, idempotência, fila |
| `references/ambientes-e-chaves.md` | Sandbox, chaves de API, segurança, ida para produção |
| `references/antecipacao-e-garantia.md` | Antecipar recebíveis, Conta Escrow, e o efeito da antecipação sobre o split |
| `references/limites-e-erros.md` | Tomou 429, precisa dimensionar carga, ou está lendo um erro da API |
