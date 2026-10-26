# bdc

<img width="769" height="654" alt="image" src="https://github.com/user-attachments/assets/4c5af50d-a23b-4505-a2e7-b44fe31bbd19" />

## O diagrama divide a arquitetura em duas camadas: a roxa cuida de absorver o pico e garantir justiça, a verde cuida de nunca vender além do estoque.

Como cada requisito é atendido

Acesso massivo. CDN e WAF servem a página estática e barram robôs antes de qualquer processamento. Só quem passa pela sala de espera chega ao backend, que recebe uma vazão controlada (por exemplo, 2 mil usuários por vez) em vez de milhões de acessos simultâneos. Gateway e serviço de reserva não guardam estado, então escalam horizontalmente.
Justiça para quem tem internet lenta. A sala de espera não ordena por quem chegou primeiro ao servidor, porque isso premiaria a conexão mais rápida. Quem entra dentro de uma janela de tempo (por exemplo, 10 minutos antes da abertura) recebe uma posição sorteada. Quem chega depois da janela entra no fim da fila, em ordem de chegada. A fila libera usuários em lotes, com um token assinado e de curta duração.
Nunca vender sem estoque. O estoque fica no Redis, e o decremento é uma operação atômica (script Lua). Se o contador chegar a zero, a reserva é negada na hora, sem corrida entre requisições. A reserva vale por um TTL (por exemplo, 10 minutos). Se o pagamento não ocorrer, o ingresso volta ao estoque.
Venda só para quem vai receber. O pagamento usa chave de idempotência para evitar cobrança duplicada. O pedido segue por uma fila durável até o worker, que grava no PostgreSQL com restrição UNIQUE por ingresso (ou assento). Essa é a segunda barreira: mesmo que o Redis falhe, o banco impede a venda em duplicidade. Só depois da gravação o ingresso é emitido.

Trade-offs

O Redis é rápido, mas volátil. Por isso o PostgreSQL é a fonte da verdade, e o estoque do Redis pode ser reconstruído a partir dele.
A fila de pedidos desacopla o pagamento da gravação final. O cliente vê “pedido em processamento” por alguns segundos, mas o banco não vira gargalo no pico.
