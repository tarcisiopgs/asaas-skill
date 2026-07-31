# Split de pagamento e subcontas

É assim que o Asaas resolve marketplace: subcontas separam o dinheiro por participante, e o split distribui automaticamente o valor de uma cobrança entre carteiras.

## Subcontas

Uma subconta é uma conta Asaas completa vinculada à sua conta raiz — com clientes, cobranças, saldo e configurações próprios. É o modelo usado por marketplaces, plataformas white label, ERPs e SaaS que oferecem serviços financeiros aos seus clientes.

A criação via API retorna dois dados que você precisa guardar:

| Campo | Para que serve |
|---|---|
| `walletId` | Identificador da carteira — usado em split e em transferências entre contas Asaas |
| `apiKey` | Chave para autenticar chamadas em nome daquela subconta |

### Restrição regulatória sobre a `apiKey`

Esta é a parte que costuma surpreender no meio do projeto: a `apiKey` da subconta fica disponível **apenas durante o período de avaliação regulatória**. Para continuar usando chaves de subcontas depois disso, é preciso ou aderir ao BaaS do Asaas, ou tratar-se de subcontas de filiais (mesmo prefixo de CNPJ da conta principal).

Novos clientes que criam subcontas via API passam por esse período de avaliação, com limites definidos em operações de subconta e cobrança.

O impacto no desenho é real: uma arquitetura que depende de chamar a API com a chave de cada subconta precisa confirmar, antes de construir, sob qual regime vai operar. Consulte `docs/faq-periodo-de-avaliacao.md` e valide com o Asaas — não presuma que a chave de subconta estará disponível indefinidamente.

### Configuração não é toda herdada

Subcontas herdam parte das configurações administrativas e comerciais da raiz, mas algumas precisam ser feitas uma a uma:

- webhooks
- informações fiscais para emissão de nota
- outras configurações operacionais

Um erro comum é configurar o webhook na conta raiz e concluir que todas as subcontas notificam — não notificam.

A criação de subcontas pode ter tarifa própria; verifique em Configurações de Conta > Taxas.

## Split

### Onde a cobrança nasce

A cobrança é criada na conta de **quem prestou o serviço ou fez a venda**, e o split direciona a parte dos outros participantes. Se um vendedor entrega o produto e a plataforma fica com comissão, a cobrança é do vendedor e o split aponta para a plataforma.

Inverter isso — criar tudo na conta da plataforma e "repassar" ao vendedor — muda quem é o responsável pela operação perante o cliente e o fisco, além de contrariar o desenho do produto.

### Nunca inclua a própria carteira

O saldo que sobra depois de todos os splits já é creditado automaticamente a quem emitiu a cobrança. Enviar o próprio `walletId` no array de split faz a **API retornar exceção** — a criação da cobrança falha inteira.

Isso merece atenção porque o código que faz isso parece razoável à primeira vista: "quero 5% para a plataforma, então mando o walletId da plataforma com 5%". Só funciona se a cobrança pertencer a outra conta.

### A base de cálculo é o líquido

O percentual incide sobre o `netValue` — o valor da cobrança menos a tarifa do Asaas. Não sobre o bruto.

Cobrança de R$ 100 no boleto, tarifa de R$ 2 → líquido R$ 98. Um split de 50% transfere R$ 49.

Projeções de receita feitas sobre o valor bruto vão divergir do extrato todo mês, e a diferença cresce com o volume.

### Fixo, percentual, ou os dois

- `fixedValue` — valor absoluto, até **2 casas decimais** (ex.: `9.32`)
- `percentualValue` — percentual, até **4 casas decimais** (ex.: `92.3444`)

Podem ser combinados na mesma cobrança, sem regra de prioridade. O teto é o líquido: a soma dos splits fixos não pode ultrapassá-lo, e os percentuais não podem passar de 100%.

Exemplo que a API rejeita: líquido de R$ 98, split fixo de R$ 50 mais split percentual de 50% → 49 + 50 = R$ 99, acima do líquido.

Não há limite de quantidade de carteiras no split.

### Estados

`PENDING`, `AWAITING_CREDIT`, `CANCELLED`, `DONE`, `REFUSED`, `REFUNDED`.

Em `REFUSED` vem também `refusalReason`. Um motivo documentado é `RECEIVABLE_UNIT_AFFECTED_BY_EXTERNAL_CONTRACTUAL_EFFECT` — a unidade de recebível está comprometida por efeito de contrato.

Cobranças usadas como garantia em operação de crédito, mesmo em outra instituição, não aceitam split.

### Bloqueio por divergência

Se no momento do recebimento (ou da antecipação) a soma dos splits ultrapassar o líquido disponível, o valor e o split são bloqueados e você recebe o evento `PAYMENT_SPLIT_DIVERGENCE_BLOCK`. Há **2 dias úteis** para ajustar.

Ajustou dentro do prazo e o novo total cabe no bloqueado → desbloqueia e processa. Passou do prazo → expira, os splits são cancelados e chega `PAYMENT_SPLIT_DIVERGENCE_BLOCK_FINISHED`.

Trate esses dois eventos explicitamente. Sem isso, o dinheiro fica parado e ninguém percebe até alguém reclamar.

### Estorno

Estornar a cobrança estorna o split: todas as contas que receberam têm a transferência revertida.

### Split é exclusivo da API

Não dá para criar ou gerenciar split pela interface web. Sem integração própria, restam integradores (Pluga, Make) ou o plugin de WooCommerce.

## Recuperar um `walletId` existente

Se a conta não foi criada via API ou o `walletId` não foi armazenado, existe endpoint de recuperação — precisa da chave de API da conta destino. Veja `reference/recuperar-walletid.md`.
