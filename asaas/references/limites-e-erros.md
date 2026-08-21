# Limites da API e leitura de erros

## Três limites diferentes, o mesmo 429

Este é o ponto que mais atrapalha diagnóstico: o Asaas tem **três** mecanismos de limitação independentes, e os três respondem `HTTP 429 Too Many Requests`. Receber 429 não diz qual deles estourou, e a correção é diferente em cada caso.

| Limite | Como funciona | O que fazer |
|---|---|---|
| **Rate limit** | por endpoint, nos que sofreriam com abuso | ler os headers e desacelerar |
| **Cota** | 25.000 requisições por conta a cada 12h, qualquer endpoint | reduzir volume total ou reagendar a carga |
| **Concorrência** | até 50 requisições `GET` simultâneas | limitar o paralelismo do cliente |

**Os headers distinguem o primeiro caso**, e só ele:

```
RateLimit-Limit: 100
RateLimit-Remaining: 50
RateLimit-Reset: 30
```

`RateLimit-Reset` são os **segundos** que faltam para a janela reiniciar. Quando `RateLimit-Remaining` chega a zero, o próximo 429 é de rate limit. Se você tomou 429 e esses headers não indicam esgotamento, olhe para os outros dois — provavelmente é cota ou concorrência.

**A janela da cota é deslizante a partir do seu primeiro uso.** O contador começa na primeira requisição e roda por 12 horas, então não existe "vira à meia-noite": a hora de reset depende de quando você começou. Uma carga noturna que hoje passa pode estourar amanhã só por ter iniciado mais cedo.

**Concorrência não é volume.** São requisições enviadas antes de a anterior responder. Um script que dispara 200 `GET` em paralelo estoura o teto de 50 mesmo consumindo pouquíssimo da cota diária. A correção é um pool limitado no cliente, não um `sleep` entre lotes.

### Consequência prática

Retry cego com backoff resolve o rate limit e **piora** os outros dois: em cota, tentar de novo só gasta mais do orçamento de 12h; em concorrência, o retry adiciona mais uma requisição simultânea à fila que já estourou. Antes de reagir a um 429, leia os headers para saber com qual dos três você está lidando.

## Prefira webhook a polling

O caminho mais eficaz de não bater nos limites é não perguntar em laço. O Asaas documenta a comparação em `docs/polling-vs-webhooks`, e o desenho recomendado é o de `webhooks.md`: recebe o evento, persiste, processa assíncrono.

Polling em cima de cobranças é o padrão que mais consome cota sem entregar nada — o estado só muda quando o pagador age, e o webhook avisa no instante em que isso acontece.

## Variantes `lean`

A especificação OpenAPI expõe, para boa parte das operações de cobrança, uma variante sob `/v3/lean/...` (`/v3/lean/payments`, `/v3/lean/payments/{id}`, `/v3/lean/payments/{id}/refund`, entre outras). A diferença declarada na spec é que a resposta traz **dados resumidos** em vez do objeto completo.

Quando você só precisa confirmar que a operação deu certo e guardar o `id`, a variante enxuta reduz o payload trafegado e processado. Não muda cota nem rate limit — o custo continua sendo uma requisição — então é otimização de banda e de parsing, não de limite.

Como essa família não tem página dedicada na documentação narrativa, confira o schema exato pela referência do endpoint antes de trocar uma chamada existente por ela.

## Códigos de resposta

A tabela completa está em `reference/codigos-http-das-respostas` na doc oficial. Os que mais aparecem numa integração:

- **401** — chave inválida, ausente, ou o `$` inicial comido pelo parser do `.env` (veja `ambientes-e-chaves.md`)
- **403** — a chave é válida mas não tem permissão para o recurso; em subconta, revise a restrição regulatória descrita em `split-e-subcontas.md`
- **429** — um dos três limites acima
- **400** — payload inválido; a resposta traz `errors[]` com `code` e `description`, e vale logar os dois

Erro do Asaas vem com descrição em português apontando o campo. Repasse essa descrição ao usuário em vez de um "falha ao criar cobrança" genérico — ela costuma ser suficiente para o próprio usuário corrigir.
