# Webhooks

## O contrato: at-least-once

O Asaas garante que o evento chega **ao menos uma vez**. O mesmo evento pode chegar repetido — tipicamente quando seu endpoint não responde a tempo e o Asaas assume falha na entrega.

Isso não é defeito, é o contrato. Integrações que assumem entrega única acabam duplicando pedidos, e-mails, liberação de acesso e crédito de saldo.

Os eventos chegam por `POST`, que é o único verbo HTTP sem idempotência natural — daí a responsabilidade ficar do lado de quem recebe.

## O padrão que resolve

Cada evento traz um `id` que permanece o mesmo entre reenvios. Use-o como chave de unicidade.

1. Recebeu o evento → persiste `id` + payload com restrição `UNIQUE`, status `PENDING`
2. Persistiu → responde `200` imediatamente
3. Processa a regra de negócio depois, de forma assíncrona
4. Terminou → marca `DONE` (ou remove o registro)

Violação de unicidade significa "já vi este evento": responda `200` e não processe de novo.

```sql
CREATE TABLE asaas_events (
    id              bigint PRIMARY KEY,
    asaas_event_id  text UNIQUE NOT NULL,
    payload         json NOT NULL,
    status          text NOT NULL   -- PENDING | DONE
);
```

```js
app.post("/asaas/webhooks", express.json(), async (req, res) => {
  try {
    await db.query(
      "INSERT INTO asaas_events (asaas_event_id, payload, status) VALUES ($1, $2, 'PENDING')",
      [req.body.id, req.body],
    );
  } catch (e) {
    if (e.code === "23505") return res.json({ received: true }); // duplicata: já registrado
    throw e;
  }
  return res.json({ received: true });
});
```

## O erro caro: responder 200 antes de persistir

Responder `200` e só então gravar significa que uma falha entre as duas coisas perde o evento em definitivo — o Asaas já considerou entregue e **não garante reenvio automático**.

A ordem importa: persiste, confirma, aí responde.

## Por que processar assíncrono

Processar a regra de negócio dentro do request tem dois problemas: o tempo de resposta cresce e aumenta a chance de o Asaas considerar a entrega falha (gerando reenvio), e uma exceção no meio do processamento pode devolver erro para um evento que já foi parcialmente aplicado.

Persistir e processar em worker separa as duas preocupações. Para volume alto (centenas de milhares de eventos/dia), uma fila dedicada — SQS, RabbitMQ, Kafka — em vez de tabela.

## Ordem

A entrega não garante ordem. Se a sequência importa para o seu domínio, processe a fila em ordem ascendente de recebimento e trate o caso de um evento posterior chegar antes do anterior.

## Alternativa mais simples

Se o volume é baixo e o processamento é rápido, dá para processar dentro do request e apenas registrar os `id`s já processados numa tabela de controle — consultando antes de processar. É menos robusto (o processamento continua preso ao timeout do request), mas resolve a duplicidade.

## Operação

- Webhook é configurado **por conta**. Em cenários com subcontas, cada subconta precisa da sua própria configuração — configurar só na raiz não faz as subcontas notificarem.
- Tokens de webhook têm validação de complexidade e geração automática obrigatória em versões recentes; confira o changelog da doc.
- Filas com muitas falhas consecutivas podem ser penalizadas ou pausadas. Monitore e saiba reativar: `docs/fila-pausada.md`, `docs/penalização-de-filas.md`, `docs/como-reativar-fila-interrompida.md`.
- O Asaas publica a lista de IPs oficiais, útil para allowlist: `docs/ips-oficiais-do-asaas.md`.
- Configure alerta para ausência de eventos esperados. Silêncio em fluxo de pagamento raramente significa que está tudo bem.

## Eventos de split

Além dos eventos de cobrança, trate explicitamente:

- `PAYMENT_SPLIT_DIVERGENCE_BLOCK` — soma dos splits passou do líquido; 2 dias úteis para ajustar
- `PAYMENT_SPLIT_DIVERGENCE_BLOCK_FINISHED` — prazo expirou, splits cancelados, valor liberado

Lista completa: `docs/eventos-de-webhooks.md`.
