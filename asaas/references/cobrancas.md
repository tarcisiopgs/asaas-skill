# Cobranças

Uma cobrança é um valor a receber vinculado a um cliente, com um ou mais meios de pagamento disponibilizados.

## Fluxo

```
Criar cliente  →  guardar o ID  →  criar cobrança  →  entregar a fatura  →  acompanhar status  →  conciliar
```

O cliente precisa existir antes da cobrança. O ID retornado na criação vai no campo `customer` e deve ser persistido — recriar cliente a cada cobrança gera duplicatas na base do Asaas e atrapalha a conciliação.

## Campos principais

| Campo | Observação |
|---|---|
| `customer` | ID do cliente no Asaas |
| `billingType` | Meio de pagamento |
| `value` | **Reais decimais**, não centavos: `100.50` = R$ 100,50 |
| `dueDate` | Formato `AAAA-MM-DD` |
| `externalReference` | Seu identificador interno — use sempre, é o que salva a conciliação |
| `description` | Aparece para o cliente na fatura |

### `value` é a armadilha mais cara

Sistemas que armazenam dinheiro em centavos (prática correta, evita float) precisam converter na fronteira. Mandar o inteiro cru cobra 100× o valor.

O erro não estoura: a API aceita, a cobrança é criada, o cliente recebe. Deixe a conversão explícita e nomeada no código, não embutida numa expressão.

## `billingType`

| Valor | Meio |
|---|---|
| `PIX` | Pix |
| `BOLETO` | Boleto bancário |
| `CREDIT_CARD` | Cartão de crédito |
| `DEBIT_CARD` | Cartão de débito |
| `UNDEFINED` | O cliente escolhe na tela de pagamento |

`UNDEFINED` costuma ser a melhor escolha quando você entrega a fatura hospedada e não quer decidir pelo cliente — a conversão tende a ser melhor do que forçar um meio.

## Pix

Retorna QR Code e o código copia-e-cola na resposta da cobrança. Confirmação é quase imediata, e chega por webhook — não fique fazendo polling.

Para assinatura recorrente paga em Pix, veja os fluxos de Pix Automático na doc (`docs/automatic-pix-webhook-flows.md`), que dependem de mandato autorizado pelo cliente no banco dele.

## Cartão de crédito

Duas formas de enviar os dados:

- `creditCard` + `creditCardHolderInfo` — dados completos do cartão e do portador
- `creditCardToken` — token de uma cobrança anterior, para não retransitar o número

Envie `remoteIp` (IP real do comprador) — faz parte da análise antifraude.

**Tokenize.** A primeira cobrança retorna `creditCardToken`; guarde-o e use nas seguintes. Isso reduz drasticamente seu escopo de PCI-DSS, porque o número do cartão deixa de trafegar e de ser armazenado por você. Veja `docs/pci-dss-1.md`.

### Parcelamento

O Asaas parcela em cartão de crédito, com faixas de tarifa diferentes por número de parcelas. Se o seu ticket é alto, parcelamento costuma ser condição para a venda acontecer no mercado brasileiro.

A tarifa percentual incide sobre o valor total da venda, e o valor fixo é cobrado uma única vez, não por parcela. Confirme as faixas vigentes em `asaas.com/precos-e-taxas` — elas mudam, e o contrato de cada conta pode ter condições próprias.

## Assinaturas

Recorrência gerenciada pelo Asaas: você cria a assinatura com `cycle` (`MONTHLY`, `YEARLY`, entre outros) e `nextDueDate`, e ele gera as cobranças automaticamente.

Acompanhe pelos webhooks de cobrança — cada ciclo gera uma cobrança nova com seu próprio evento. A assinatura é o contrato; a cobrança é o que efetivamente é pago.

Se o seu produto precisa de controle fino sobre quando e quanto cobrar (uso variável, upgrade no meio do ciclo, proporcionalidade), avalie gerar cobranças avulsas pela sua própria lógica em vez de usar assinatura.

## Status e conciliação

Acompanhe por webhook, não por polling. Guarde o `id` da cobrança do Asaas junto do seu registro interno, e use `externalReference` no sentido inverso — assim a conciliação funciona a partir de qualquer um dos dois lados quando algo dá errado.

## Alternativas à cobrança via API

- **Link de Pagamento** — link pronto, sem integração de checkout
- **Checkout** — tela hospedada com resumo da compra, aceita split

Valem quando você não quer construir a tela de pagamento. Veja `docs/link-de-pagamentos.md` e `docs/checkout-asaas.md`.

## Antes de reenviar uma requisição que falhou

Timeout ou erro de rede não significa que a operação não aconteceu. Consulte seus registros (ou a API, por `externalReference`) antes de repetir — reenvio cego é a origem clássica de cobrança duplicada.
