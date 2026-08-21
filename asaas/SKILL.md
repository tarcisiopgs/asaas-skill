---
name: asaas
description: Integração com a API do Asaas — gateway de pagamentos e conta digital brasileiro. Cobrir cobranças (Pix, boleto, cartão de crédito, parcelamento), assinaturas recorrentes, Pix Automático, split de pagamento, subcontas para marketplace, antecipação de recebíveis, Conta Escrow, webhooks, limites da API e o ambiente Sandbox. Use sempre que aparecer "Asaas", `api.asaas.com`, `access_token`, `walletId`, `billingType`, `$aact_`, ou quando a tarefa envolver Pix, boleto, split de pagamento, subconta, antecipar recebível, reter valor em garantia, ou marketplace brasileiro que repassa dinheiro a terceiros — mesmo que o gateway não seja nomeado. Use também ao diagnosticar 429 ou erro de autenticação no Asaas, para escolher entre Asaas e outro gateway para um produto no Brasil, e antes de consultar a documentação do Asaas por WebFetch ou curl.
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

### 3. A chave começa com `$` — e isso quebra arquivos `.env`

As chaves têm o formato `$aact_prod_...` (produção) ou `$aact_hmlg_...` (Sandbox). O `$` inicial é interpretado como expansão de variável por praticamente todo parser de `.env` e por shells, o que silenciosamente transforma a chave em string vazia — e você recebe 401 sem entender por quê.

Escape conforme o parser em uso. Aspas simples nem sempre bastam: alguns runtimes expandem `$` mesmo dentro delas. A combinação que costuma satisfazer tanto o parser da aplicação quanto o Docker Compose é aspas duplas com barra invertida:

```bash
ASAAS_API_KEY="\$aact_prod_000..."
```

Depois de mudar o `.env`, reinicie o processo — watchers de arquivo geralmente não recarregam variáveis de ambiente.

**O prefixo também revela o ambiente.** `$aact_hmlg_` é Sandbox e `$aact_prod_` é produção. Vale uma verificação na inicialização: se o prefixo da chave não combina com a URL base configurada, aborte com uma mensagem clara em vez de deixar a aplicação apontar para produção com credencial de teste (ou o contrário, o que é pior — cobranças reais durante um teste).

### 4. Sandbox e produção têm hosts diferentes

| Ambiente | URL base |
|---|---|
| Produção | `https://api.asaas.com/v3` |
| Sandbox | `https://api-sandbox.asaas.com/v3` |

As chaves não são intercambiáveis entre ambientes. Se encontrar código apontando para `sandbox.asaas.com/api/v3`, é o host antigo — confirme na doc antes de manter.

## Split de pagamento

Split é como o Asaas resolve marketplace: a cobrança nasce em uma conta e parte do valor é creditada automaticamente em outras. Três regras concentram quase todos os bugs.

**A cobrança pertence a quem prestou o serviço.** Se o vendedor é quem entrega, a cobrança é criada na conta dele e o split direciona a comissão para a plataforma. Criar tudo na conta da plataforma inverte o fluxo do dinheiro e muda quem é o responsável pela operação perante o cliente e o fisco.

Isso implica autenticar como a subconta — e o acesso à chave de API de subcontas tem restrição regulatória que precisa ser confirmada antes de desenhar a arquitetura. Leia `references/split-e-subcontas.md` antes de assumir que essa chave estará disponível.

**Nunca inclua a própria carteira no split.** O que sobra depois dos splits já é creditado automaticamente a quem emitiu a cobrança. Mandar o próprio `walletId` faz a API retornar exceção — não é um no-op silencioso, é um erro que derruba a criação da cobrança.

**O percentual incide sobre o líquido, não sobre o bruto.** A base de cálculo é o `netValue`, que é o valor da cobrança menos a tarifa do Asaas. Uma cobrança de R$ 100 com tarifa de R$ 2 tem R$ 98 de base — um split de 50% transfere R$ 49, não R$ 50. Modelos de receita construídos sobre o bruto vão divergir da conta bancária todo mês.

Estados, bloqueio por divergência e limites de casas decimais: `references/split-e-subcontas.md`.

## Webhooks

O Asaas entrega eventos com garantia **at-least-once**: o mesmo evento pode chegar mais de uma vez, especialmente quando seu endpoint demora a responder. Isso não é falha, é o contrato — e integrações que assumem entrega única duplicam pedidos, e-mails e liberações de acesso.

O padrão que resolve: cada evento traz um `id` estável entre reenvios. Persista esse `id` com restrição de unicidade, responda `200` assim que a persistência confirmar, e processe a regra de negócio depois, de forma assíncrona. Violação de unicidade significa "já vi esse evento" — responda `200` e siga em frente.

Responder `200` antes de persistir é a falha mais cara: o Asaas considera entregue e você perdeu o evento. Detalhes, exemplo de esquema e tratamento de fila pausada: `references/webhooks.md`.

## Depois que o dinheiro entra

Duas funcionalidades mexem em **quando** o valor fica disponível, e as duas afetam conciliação.

**Antecipação** adianta o recebimento cobrando uma taxa a mais. O detalhe que quebra integração de marketplace: se a cobrança tem split, a antecipação **muda a base de cálculo do repasse**, porque o líquido passa a descontar também a taxa de antecipação. Em split fixo a antecipação é recusada quando não cabe — falha visível. Em split percentual ela prossegue e o parceiro simplesmente recebe menos do que a conta feita sobre o bruto. Ninguém é avisado.

**Conta Escrow** faz o contrário: retém o que a subconta recebe até a liberação. Enquanto retido, o recebimento aconteceu mas o saldo não existe — e uma plataforma que trata "cobrança recebida" como "dinheiro disponível" diverge do extrato o tempo todo.

Detalhes, regras e exemplos: `references/antecipacao-e-garantia.md`.

## Três limites diferentes devolvem o mesmo 429

O Asaas limita por **rate limit** (por endpoint, com headers `RateLimit-*`), por **cota** (25.000 requisições por conta a cada 12h) e por **concorrência** (até 50 `GET` simultâneos). Os três respondem `429`, então o código sozinho não diz qual estourou.

Isso importa porque a reação é oposta: retry com backoff resolve o rate limit, mas em cota só queima mais do orçamento de 12h, e em concorrência adiciona mais uma requisição à fila que já transbordou. Leia os headers antes de reagir. Detalhes em `references/limites-e-erros.md`.

## Escolhendo entre Asaas e um gateway internacional

Para produtos que cobram no Brasil, a comparação raramente é sobre percentual de taxa. Os fatores que costumam decidir:

- **Pix** é obrigatório na prática no varejo brasileiro. Confirme se o concorrente oferece Pix para contas brasileiras sem lista de espera, e se cobre Pix recorrente.
- **Parcelamento no cartão** é expectativa de mercado em ticket alto. Nem todo gateway internacional oferece parcelamento no Brasil.
- **Marketplace com repasse a terceiros** exige subcontas com KYC e split. Verifique se o concorrente permite onboarding self-service dos recebedores, ou se cada um precisa de aprovação individual.
- **Boleto e NF-e** existem no Asaas nativamente e costumam ser produto separado (ou inexistente) fora do Brasil.

Não decida por tabela de preço isolada: um gateway com taxa menor que não entrega Pix ou parcelamento não é mais barato, é inviável. Confirme cada ponto na documentação vigente dos dois lados antes de recomendar — disponibilidade regional muda com frequência.

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
