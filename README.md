# BigDataCorp Desenho Arquitetura
## Rock in Rio - Arquitetura de Vendas de Alta Escala
Este projeto descreve a arquitetura de software desenhada para sustentar a venda de ingressos de eventos de alto tráfego, priorizando a justiça (FIFO), disponibilidade e integridade do inventário.

## Diagrama da Arquitetura:
<img width="1909" height="504" alt="image" src="https://github.com/user-attachments/assets/ffd7fb28-5602-4f7d-ad30-2323d3985720" />

## Componentes Principais:
1. CDN (Edge): Atua como a primeira linha de defesa, armazenando arquivos estáticos e reduzindo a latência global. Protege o front-end contra picos de tráfego de leitura.
2. Load Balancer: Distribui o tráfego de entrada para os serviços de API, garantindo alta disponibilidade.
3. API Gateway & Sala de Espera: Orquestra o tráfego. A Sala de Espera implementa uma fila lógica (FIFO) para garantir que usuários com conexões lentas tenham sua posição preservada, evitando o efeito de "fura-fila".
4. Auth Service: Gerencia a identidade do usuário de forma segura antes da permissão de compra.
5. Ticket API & Cache (Redis): O coração da venda. A API utiliza o Redis como "Fonte Única da Verdade" para o inventário, permitindo validações atômicas de estoque sem sobrecarregar o banco de dados.
6. Mensageria (Message Broker): Desacopla o processo de venda. A API apenas valida e enfileira o pedido, workers assíncronos processam o pagamento e a emissão do ticket, garantindo que o sistema não trave sob carga pesada.
7. Workers (Pagamento e Ticket): Processam tarefas em segundo plano. Se o serviço de pagamento oscilar, as transações permanecem seguras na fila até serem processadas com sucesso, se houver aumento de demanda pode ser configurado escala horizontal aumentando a disponibilidade.
8. SQL Database: Atua como o sistema de registro final, persistindo os dados de forma consistente e auditável(ACID).

## Estratégias de Alta Performance:
1. Pré-aquecimento do Inventário (Warm-up)
Para evitar o "Efeito de Manada" (Thundering Herd), um processo administrativo pré-carrega o estoque do banco de dados para o Cache (Redis) minutos antes da abertura das vendas. A Ticket API interage exclusivamente com o Cache durante o pico.

2. Atomicidade e Consistência
A contagem de ingressos é realizada via decremento atômico no Redis (DECR). Isso garante que, mesmo sob concorrência extrema, o número exato de ingressos vendidos nunca exceda a disponibilidade real, evitando o overselling.

3. Tolerância a Falhas
A arquitetura é baseada em eventos. Caso o Ticket Worker falhe em persistir um ingresso no SQL, o uso de filas de mensagens permite o reprocessamento automático (retry), garantindo que nenhum cliente perca sua compra por uma falha momentânea de rede ou de banco de dados.
