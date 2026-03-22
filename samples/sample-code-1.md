# 샘플 코드 — 출금 요청 처리

아래 코드를 리뷰 대상으로 사용합니다.

```java
@Service
public class WithdrawService {

    @Autowired
    private WalletRepository walletRepository;

    @Autowired
    private TransactionRepository transactionRepository;

    @Autowired
    private NotificationService notificationService;

    public WithdrawResponse withdraw(Long userId, double amount, String address) {
        // 잔액 확인
        Wallet wallet = walletRepository.findByUserId(userId);
        if (wallet.getBalance() < amount) {
            throw new RuntimeException("잔액 부족");
        }

        // 출금 처리
        wallet.setBalance(wallet.getBalance() - amount);
        walletRepository.save(wallet);

        // 트랜잭션 기록
        Transaction tx = new Transaction();
        tx.setUserId(userId);
        tx.setAmount(amount);
        tx.setAddress(address);
        tx.setType("WITHDRAW");
        tx.setStatus("PENDING");
        tx.setCreatedAt(new Date());
        transactionRepository.save(tx);

        // 알림 발송
        notificationService.send(userId, "출금 요청이 접수되었습니다. 금액: " + amount + ", 주소: " + address);

        return new WithdrawResponse(tx.getId(), "SUCCESS");
    }

    public List<Transaction> getHistory(Long userId, String type) {
        if (type != null) {
            return transactionRepository.findByUserIdAndType(userId, type);
        }
        return transactionRepository.findByUserId(userId);
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 출금 서비스
- 동시에 여러 출금 요청이 들어올 수 있음
- 규제 요건: 일일 출금 한도, 본인 인증 필요
- 알림에 민감 정보가 포함되면 안 됨
