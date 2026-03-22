# 샘플 코드 2 — 자동 매수 예약 처리

```java
@Service
@RequiredArgsConstructor
public class AutoBuyService {

    private final OrderRepository orderRepository;
    private final PriceService priceService;
    private final WalletService walletService;
    private final UserService userService;

    public AutoBuyResponse createReservation(Long userId, String symbol,
                                              BigDecimal targetPrice, BigDecimal quantity) {
        List<Order> existing = orderRepository.findByUserIdAndStatus(userId, OrderStatus.WAITING);
        if (existing.size() >= 10) {
            throw new BusinessException("최대 예약 건수를 초과했습니다");
        }

        BigDecimal estimatedCost = targetPrice.multiply(quantity);
        BigDecimal balance = walletService.getBalance(userId);
        if (balance.compareTo(estimatedCost) < 0) {
            throw new InsufficientBalanceException("예상 체결 금액 대비 잔액이 부족합니다");
        }

        Order order = Order.builder()
            .userId(userId)
            .symbol(symbol)
            .targetPrice(targetPrice)
            .quantity(quantity)
            .type(OrderType.AUTO_BUY)
            .status(OrderStatus.WAITING)
            .build();
        orderRepository.save(order);

        return AutoBuyResponse.of(order);
    }

    @Scheduled(fixedRate = 5000)
    public void checkAndExecute() {
        List<Order> waitingOrders = orderRepository.findByStatus(OrderStatus.WAITING);

        for (Order order : waitingOrders) {
            try {
                BigDecimal currentPrice = priceService.getCurrentPrice(order.getSymbol());

                if (currentPrice.compareTo(order.getTargetPrice()) <= 0) {
                    executeOrder(order, currentPrice);
                }
            } catch (Exception e) {
                log.warn("주문 체결 실패 - orderId: {}, error: {}", order.getId(), e.getMessage());
            }
        }
    }

    @Transactional
    private void executeOrder(Order order, BigDecimal executedPrice) {
        BigDecimal totalCost = executedPrice.multiply(order.getQuantity());
        walletService.deduct(order.getUserId(), totalCost);

        order.execute(executedPrice);
        orderRepository.save(order);

        log.info("자동 매수 체결 - symbol: {}, quantity: {}, price: {}, userId: {}",
            order.getSymbol(), order.getQuantity(), executedPrice, order.getUserId());
    }

    public void cancelReservation(Long orderId, Long userId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new BusinessException("주문을 찾을 수 없습니다"));

        if (!order.getUserId().equals(userId)) {
            throw new UnauthorizedException("본인의 주문만 취소할 수 있습니다");
        }

        order.cancel();
        orderRepository.save(order);
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 자동 매수 예약 기능
- 5초마다 가격을 확인하여 목표가 이하이면 자동 체결
- 동시에 수천 개의 예약이 활성화될 수 있음
- 급격한 가격 변동 시 슬리피지(체결가와 예상가 차이) 발생 가능
- 규제: 주문 실행 전 본인 인증 상태 확인 필요
- 사용자당 최대 예약 건수 제한 있음
