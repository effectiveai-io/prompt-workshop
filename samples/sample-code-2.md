# 샘플 코드 2 — 자동 매수 예약 처리

```java
@Service
public class AutoBuyService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private PriceService priceService;

    @Autowired
    private WalletService walletService;

    public AutoBuyResponse createReservation(Long userId, String symbol, double targetPrice, double quantity) {
        // 예약 생성
        Order order = new Order();
        order.setUserId(userId);
        order.setSymbol(symbol);
        order.setTargetPrice(targetPrice);
        order.setQuantity(quantity);
        order.setType("AUTO_BUY");
        order.setStatus("WAITING");
        order.setCreatedAt(new Date());
        orderRepository.save(order);

        return new AutoBuyResponse(order.getId(), "RESERVED");
    }

    @Scheduled(fixedRate = 5000)
    public void checkAndExecute() {
        List<Order> waitingOrders = orderRepository.findByStatus("WAITING");

        for (Order order : waitingOrders) {
            double currentPrice = priceService.getCurrentPrice(order.getSymbol());

            if (currentPrice <= order.getTargetPrice()) {
                // 매수 실행
                double totalCost = currentPrice * order.getQuantity();
                walletService.deduct(order.getUserId(), totalCost);

                order.setStatus("EXECUTED");
                order.setExecutedPrice(currentPrice);
                order.setExecutedAt(new Date());
                orderRepository.save(order);

                System.out.println("자동 매수 체결: " + order.getSymbol() +
                    " 수량: " + order.getQuantity() +
                    " 가격: " + currentPrice +
                    " 사용자: " + order.getUserId());
            }
        }
    }

    public void cancelReservation(Long orderId, Long userId) {
        Order order = orderRepository.findById(orderId).get();
        if (order.getUserId() == userId) {
            order.setStatus("CANCELLED");
            orderRepository.save(order);
        }
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 자동 매수 예약 기능
- 5초마다 가격을 확인하여 목표가 이하이면 자동 체결
- 동시에 수천 개의 예약이 활성화될 수 있음
- 급격한 가격 변동 시 슬리피지(체결가와 예상가 차이) 발생 가능
- 규제: 주문 실행 전 본인 인증 상태 확인 필요
