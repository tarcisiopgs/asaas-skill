# Antecipação e Conta Escrow

Duas funcionalidades que mexem em **quando** o dinheiro fica disponível — e em direções opostas. A antecipação adianta o recebimento cobrando por isso; a Conta Escrow segura o valor até uma condição comercial ser satisfeita. As duas mudam a conciliação financeira da sua plataforma, e nenhuma delas falha de forma barulhenta.

## Antecipação

Antecipar troca prazo por dinheiro: o Asaas credita agora e desconta uma taxa. Funciona para cobrança avulsa (campo `payment`) ou parcelamento (campo `installment`).

```http
POST /v3/anticipations
{ "payment": "pay_626366773834" }
```

Uma diferença por meio de pagamento que muda o desenho da integração: em parcelamento no **cartão**, dá para antecipar o parcelamento inteiro **ou** parcela a parcela; em parcelamento no **boleto**, é obrigatoriamente parcela a parcela. Código que assume o modo "tudo de uma vez" quebra quando o cliente escolhe boleto.

**Simule antes de solicitar** (`POST /v3/anticipations/simulate`). Além do custo, a simulação devolve `isDocumentationRequired`, que diz se aquela antecipação exige envio de nota fiscal ou contrato de prestação de serviço. Descobrir isso só na hora da solicitação trava o fluxo com o cliente esperando.

Vale a leitura econômica: no Asaas a taxa de cartão é plana ao longo do parcelamento, então **parcelar é barato e antecipar é que custa**. Se a régua for margem, a decisão de antecipar merece ser consciente e não um padrão ligado no código.

Ainda: `GET /v3/payments/{id}/contractualEffectRestrictions` revela efeitos de contrato que bloqueiam a antecipação daquela cobrança — útil quando a solicitação é recusada e não se sabe por quê.

### A interação que quebra silenciosamente: antecipação com split

Esta é a armadilha cara da seção. Se a cobrança tem split configurado, **a antecipação muda a base de cálculo do split**, porque o valor líquido final passa a descontar também a taxa de antecipação.

A regra vale conforme o tipo de split, e o efeito é bem diferente:

| Tipo | O que acontece | Como falha |
|---|---|---|
| **Fixo** | o valor é validado contra o líquido final após taxas do Asaas **e** de antecipação | se exceder, **a antecipação não prossegue** — falha visível |
| **Percentual** | o percentual é calculado sobre o líquido final após antecipação | a antecipação prossegue e **o parceiro recebe menos** do que a conta feita sobre o bruto — falha invisível |

O exemplo da própria documentação deixa claro:

```
Valor bruto:                        R$ 100,00
Líquido após taxas + antecipação:   R$  90,00

Split fixo de R$ 95,00   → antecipação recusada
Split de 50%             → repassa R$ 45,00 (não R$ 50,00)
```

O caso percentual é o perigoso justamente porque nada estoura: você prometeu metade de cem ao parceiro, repassou quarenta e cinco, e a diferença só aparece quando ele reclama. Se a sua plataforma comunica ao parceiro um valor calculado sobre o bruto, ela está errada sempre que houver antecipação.

**Nunca use o valor bruto da cobrança como base para validar o que será repassado.** Vale para split em geral (veja `split-e-subcontas.md`), e vale em dobro aqui, porque a antecipação adiciona uma segunda camada de taxa sobre a mesma cobrança.

## Conta Escrow

O contrário: retém o valor recebido por uma subconta até a liberação. Serve a marketplace que só quer liberar o pagamento depois da entrega, plataforma de serviço que espera a conclusão, ou qualquer operação com período de garantia.

```
Habilitar escrow na subconta
↓  cobranças recebidas ficam bloqueadas (não entram no saldo disponível)
↓  liberação automática na expiração  ou  POST /v3/escrow/{id}/finish
↓  valor passa a compor o saldo da subconta
```

A configuração é **por subconta** (`POST /v3/accounts/{id}/escrow`), com um padrão possível para todas (`POST /v3/accounts/escrow`). Subconta sem o recurso habilitado recebe normalmente, sem retenção.

Quatro regras que decidem o desenho da integração:

**Só vale para o futuro.** Apenas cobranças recebidas **após** a habilitação entram no fluxo de retenção. Ligar o escrow não segura o que já caiu — quem espera efeito retroativo desenha o fluxo errado.

**Desabilitar libera tudo.** Ao desligar o escrow de uma subconta, todos os valores ainda sob garantia são liberados de uma vez. Não é uma operação neutra de configuração: é uma liberação em massa de dinheiro.

**E não dá para voltar atrás no meio.** Enquanto existirem valores em processo de liberação, a configuração não pode ser reabilitada para aquela subconta. Desligar por engano custa uma janela de espera, não um clique.

**A conciliação precisa saber disso.** Durante a retenção, o recebimento aconteceu mas o saldo não está disponível. Uma plataforma que trata "cobrança recebida" como "dinheiro disponível" vai divergir do extrato o tempo todo. Consulte a garantia por `GET /v3/payments/{id}/escrow`.

## Onde isso aparece nos webhooks

Antecipações emitem eventos próprios — a doc os lista em `docs/webhook-para-antecipacoes`. Se a sua conciliação depende de antecipação, assine esses eventos em vez de perguntar em laço pelo status; o raciocínio de idempotência de `webhooks.md` vale igual aqui.
