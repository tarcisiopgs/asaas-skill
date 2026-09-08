# Webhooks

## Recepção e deduplicação

O Asaas utiliza entrega **at-least-once**: eventos podem se repetir com o mesmo `id`. Não dependa de entrega única.

1. Valide `asaas-access-token` contra o `authToken` configurado para este webhook, antes de persistir. Use um segredo próprio, nunca a API key do Asaas.
2. Valide o payload e insira `id` + payload atomicamente com unicidade e status `PENDING`.
3. Confirme a persistência e então responda `200`, inclusive para duplicatas já persistidas.
4. Processe em worker e marque `DONE`. Preserve o identificador para impedir reprocessamento em reenvios futuros; remover o payload por política de retenção não deve apagar essa proteção.

Exemplo ilustrativo para uma integração/conta, em PostgreSQL e Express. Pressupõe `app`, um pool `db` sem transação aberta (autocommit) e configuração de segredo carregada pelo runtime. Se compartilhar armazenamento entre contas/ambientes, inclua esse escopo, determinado pela configuração autenticada do endpoint, na chave de unicidade.

```sql
CREATE TABLE asaas_events (
    id              bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    asaas_event_id  text UNIQUE NOT NULL,
    payload         jsonb NOT NULL,
    status          text NOT NULL CHECK (status IN ('PENDING', 'DONE'))
);
```

```js
import { timingSafeEqual } from "node:crypto";

const webhookToken = process.env.ASAAS_WEBHOOK_TOKEN;
if (!webhookToken) throw new Error("Configure o token do webhook");
const expectedToken = Buffer.from(webhookToken);

app.post("/asaas/webhooks", express.json(), async (req, res) => {
  const receivedToken = Buffer.from(req.get("asaas-access-token") || "");
  if (receivedToken.length !== expectedToken.length ||
      !timingSafeEqual(receivedToken, expectedToken)) {
    return res.sendStatus(401);
  }
  if (typeof req.body?.id !== "string" || !req.body.id.trim() ||
      typeof req.body?.event !== "string" || !req.body.event.trim()) {
    return res.sendStatus(400);
  }
  try {
    await db.query(
      `INSERT INTO asaas_events (asaas_event_id, payload, status)
       VALUES ($1, $2, 'PENDING')
       ON CONFLICT (asaas_event_id) DO NOTHING`,
      [req.body.id, req.body],
    );
    return res.status(200).json({ received: true });
  } catch {
    return res.sendStatus(503); // sem confirmação: permite nova tentativa
  }
});
```

A restrição `UNIQUE` com `ON CONFLICT` cobre entregas concorrentes. Uma consulta de existência seguida de inserção separada não oferece essa proteção. Se usar transação explícita, confirme o `COMMIT` antes do `200`; falhas de persistência não devem ser confirmadas como sucesso. Adapte a validação dos recursos do payload aos eventos assinados antes de aplicar regras de negócio.

O `catch` retorna `503` para qualquer falha do `INSERT`, inclusive persistente. Gere alertas para essas falhas: retentativas sozinhas não corrigem dados incompatíveis ou erros de armazenamento e podem [interromper a fila no Asaas](https://docs.asaas.com/docs/fila-pausada). Corrija a causa, valide a persistência do evento que falhou e, se necessário, [reative a fila](https://docs.asaas.com/docs/como-reativar-fila-interrompida) pelo painel ou pela API; acompanhe a retomada dos eventos pendentes.

O exemplo não inclui quarentena. Se precisar aceitar payloads autenticados que `jsonb` não consegue representar, preserve o conteúdo original e seu identificador em armazenamento durável compatível, com deduplicação, alerta e caminho de reprocessamento, antes de responder `200`. Sem persistência confirmada, mantenha a resposta de falha; nunca descarte o evento apenas para desbloquear a fila.

## Processamento e recuperação

Este exemplo cobre a entrada durável, não a execução da regra de negócio. Um worker precisa reivindicar cada evento atomicamente para impedir dois consumidores simultâneos e recuperar tentativas interrompidas. Para efeitos no mesmo banco, aplique a mudança de negócio e `DONE` na mesma transação, com bloqueio do registro. Uma falha deve reverter ambos para permitir retry.

Chamadas externas não participam dessa transação. Use uma outbox gravada junto da mudança local e envio com chave de idempotência quando o destino a suportar; preveja reconciliação se não suportar. Outbox sozinha não garante execução única no destino. Se publicar em broker, preserve a transição durável entre banco e fila para não perder eventos entre duas gravações.

Valide duplicatas simultâneas, token inválido, falha de persistência e retomada após falha do worker. O recebimento duplicado não deve repetir efeitos de negócio. Monitore pendências e retries: após o `200`, a recuperação passa a ser responsabilidade da aplicação.

## Ordem

O `sendType` **`SEQUENTIALLY` preserva a ordem de entrega**; `NON_SEQUENTIALLY` permite entregas paralelas sem garantia de ordem. Se o domínio depender da sequência, preserve-a também no processamento interno: workers paralelos podem concluir fora de ordem mesmo com entrega sequencial. Ordenar apenas pela chegada não reconstrói a sequência de origem no modo não sequencial.

Fontes: [introdução aos webhooks](https://docs.asaas.com/docs/sobre-os-webhooks), [idempotência](https://docs.asaas.com/docs/como-implementar-idempotencia-em-webhooks) e [criação e tipos de envio](https://docs.asaas.com/reference/criar-novo-webhook). As recomendações de transação e outbox descrevem responsabilidades da aplicação, não garantias adicionais do Asaas.

## Operação

- Webhook é configurado **por conta**. Em cenários com subcontas, cada subconta precisa da sua própria configuração — configurar só na raiz não faz as subcontas notificarem.
- Configure e armazene o `authToken` do webhook com segurança. A referência descreve geração automática quando omitido, com retorno apenas na criação; confirme o contrato vigente antes de provisionar e nunca registre esse segredo em logs.
- Filas com muitas falhas consecutivas podem ser penalizadas ou pausadas. Monitore e saiba reativar: `docs/fila-pausada.md`, `docs/penalização-de-filas.md`, `docs/como-reativar-fila-interrompida.md`.
- O Asaas publica a lista de IPs oficiais, útil para allowlist: `docs/ips-oficiais-do-asaas.md`.
- Configure alerta para ausência de eventos esperados. Silêncio em fluxo de pagamento raramente significa que está tudo bem.

## Eventos de split

Além dos eventos de cobrança, trate explicitamente:

- `PAYMENT_SPLIT_DIVERGENCE_BLOCK` — soma dos splits passou do líquido; 2 dias úteis para ajustar
- `PAYMENT_SPLIT_DIVERGENCE_BLOCK_FINISHED` — prazo expirou, splits cancelados, valor liberado

Lista completa: `docs/eventos-de-webhooks.md`.
