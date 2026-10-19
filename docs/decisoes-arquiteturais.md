# Decisões arquiteturais e tratamento de falhas

## 1. Estratégia adotada

A solução utiliza Saga por orquestração com mensagens assíncronas. O orquestrador salva o andamento da Saga em um banco próprio. O broker transporta comandos e respostas, mas não é responsável por decidir o resultado da compra.

Para não perder mensagens entre salvar um dado e publicar um evento, cada serviço utiliza Transactional Outbox. Um processo publica as mensagens da outbox e pode tentar novamente quando necessário.

## 2. Contratos das mensagens

Todas as mensagens carregam `messageId`, `sagaId`, `orderId`, `type`, `occurredAt` e `correlationId`. Os comandos também carregam `commandId` para permitir repetição segura. O serviço responde com o mesmo `sagaId` e o resultado da operação.

- `ReserveStock`: `orderId`, itens e quantidades. Resposta `StockReserved` com `reservationId`, ou `StockRejected`.
- `AuthorizePayment`: `orderId`, valor, moeda e chave de idempotência. Resposta `PaymentAuthorized` com referência do gateway, ou `PaymentRejected`.
- `ReleaseStock`: `reservationId` e motivo. Resposta `StockReleased`.
- `ConfirmOrder` e `CancelOrder`: `orderId` e motivo quando necessário. Respostas `OrderConfirmed` e `OrderCancelled`.

Dados sensíveis de cartão não devem trafegar nessas mensagens. O Payment Service trabalha com token seguro fornecido pelo gateway.

## 3. Controle de estado

A tabela `saga_state` guarda `saga_id`, `order_id`, `status`, `current_step`, `reservation_id`, `payment_reference`, `attempt_count`, `deadline_at`, `last_error` e `version`.

Estados principais: `STARTED`, `WAITING_STOCK`, `WAITING_PAYMENT`, `CONFIRMING_ORDER`, `COMPLETED`, `COMPENSATING`, `CANCELLED` e `MANUAL_REVIEW`.

O campo `version` permite atualização otimista, evitando que duas instâncias avancem a mesma Saga ao mesmo tempo. Uma tarefa periódica procura etapas com prazo vencido e faz reconciliação antes de repetir comandos.

## 4. Matriz de decisão

| Situação | Decisão do sistema |
| --- | --- |
| Estoque indisponível | Cancelar o pedido, sem tentar pagamento |
| Estoque reservado e pagamento recusado | Liberar estoque e cancelar pedido |
| Pagamento autorizado | Confirmar pedido |
| Resposta de pagamento não chegou | Consultar o status no gateway pela chave de idempotência antes de decidir |
| Mensagem recebida duas vezes | Consultar registro de processamento e devolver o resultado anterior |
| Liberação do estoque falhou temporariamente | Repetir com backoff e manter Saga em `COMPENSATING` |
| Muitas tentativas sem sucesso | Enviar para DLQ, alertar a equipe e marcar para revisão |

## 5. Idempotência

Cada consumidor guarda o `messageId` ou `commandId` em uma tabela de mensagens processadas com restrição de unicidade. O registro e a alteração de negócio acontecem na mesma transação local. Assim, se a mensagem chegar novamente, o serviço não reserva duas vezes nem libera estoque duas vezes.

No Payment Service a chave de idempotência também é enviada ao gateway, quando suportada. Em caso de timeout não se assume que o pagamento falhou, pois ele pode ter sido autorizado sem que a resposta tenha chegado.

## 6. Timeouts e tentativas

Os tempos abaixo são valores propostos para o exercício e devem ser ajustados com testes reais.

- Reserva de estoque: prazo inicial de 10 segundos.
- Autorização do pagamento: prazo inicial de 30 segundos.
- Tentativas de falhas temporárias: backoff exponencial com variação aleatória, até 3 tentativas antes de encaminhar para análise.
- Recusa definitiva do cartão: não repetir a cobrança, iniciar compensação.

Circuit Breaker impede chamadas repetidas ao gateway quando ele está instável. Retry é usado só para falhas transitórias, nunca para uma recusa definitiva. Monitoramento acompanha tempo de cada etapa, quantidade de Sagas pendentes, tamanho da DLQ e falhas de compensação.

## 7. Dead-letter queue e recuperação

Mensagens que não puderem ser processadas após as tentativas configuradas vão para uma DLQ com motivo, identificador e histórico de tentativas. A equipe recebe um alerta e pode corrigir a causa e reprocessar com segurança, respeitando a idempotência.

Uma Saga não deve ser considerada cancelada enquanto uma compensação necessária ainda estiver pendente. Se não for possível confirmar o estado do pagamento, o fluxo fica aguardando reconciliação, evitando cancelar um pedido com uma cobrança que pode ter sido concluída.

## 8. Riscos e limites

A Saga oferece consistência eventual, não isolamento completo como uma transação ACID. Para evitar venda acima do estoque disponível, a reserva deve ser atômica dentro do Inventory Service e ter controle de concorrência. Reservas temporárias precisam de prazo de expiração e uma rotina de reconciliação.

O orquestrador também pode ficar indisponível, por isso seu estado é persistido e várias instâncias podem processar Sagas com controle de concorrência. Logs com `correlationId`, métricas e rastreamento distribuído ajudam a entender os erros.

Uma compensação não é necessariamente um rollback perfeito. Se o gateway já capturou dinheiro, por exemplo, pode ser necessário estorno, que é uma operação separada e pode demorar.

## Referência

https://microservices.io/patterns/data/saga.html
