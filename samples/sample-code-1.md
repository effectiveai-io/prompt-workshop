# 샘플 코드 1 — 출금 요청 처리

```java
@Service
@RequiredArgsConstructor
public class WithdrawService {

    private final WalletRepository walletRepository;
    private final TransactionRepository transactionRepository;
    private final NotificationService notificationService;
    private final WithdrawValidator validator;

    @Transactional
    public WithdrawResponse withdraw(Long userId, BigDecimal amount, String address) {
        validator.validate(userId, amount, address);

        Wallet wallet = walletRepository.findByUserId(userId)
            .orElseThrow(() -> new BusinessException("지갑을 찾을 수 없습니다"));

        if (wallet.getBalance().compareTo(amount) > 0) {
            BigDecimal newBalance = wallet.getBalance().subtract(amount);
            wallet.updateBalance(newBalance);
        } else {
            throw new InsufficientBalanceException("잔액이 부족합니다");
        }

        Transaction tx = Transaction.builder()
            .userId(userId)
            .amount(amount)
            .address(address)
            .type(TransactionType.WITHDRAW)
            .status(TransactionStatus.PENDING)
            .fee(calculateFee(amount))
            .build();
        transactionRepository.save(tx);

        notificationService.sendWithdrawNotification(userId, tx);

        log.info("출금 요청 처리 완료 - userId: {}, amount: {}, address: {}, txId: {}",
            userId, amount, address, tx.getId());

        return WithdrawResponse.of(tx);
    }

    private BigDecimal calculateFee(BigDecimal amount) {
        BigDecimal feeRate = new BigDecimal("0.001");
        return amount.multiply(feeRate);
    }

    @Transactional(readOnly = true)
    public List<TransactionDto> getHistory(Long userId, String type, int page, int size) {
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdAt").descending());

        if (type != null) {
            return transactionRepository.findByUserIdAndType(userId, TransactionType.valueOf(type), pageable)
                .map(TransactionDto::from)
                .getContent();
        }
        return transactionRepository.findByUserId(userId, pageable)
            .map(TransactionDto::from)
            .getContent();
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 출금 서비스
- 동시에 여러 출금 요청이 들어올 수 있음
- 규제: 일일 출금 한도 확인 필요, 출금 주소 화이트리스트 확인 필요
- 수수료 정책: 출금 금액의 0.1%, 최소 수수료 있음
- 알림에 민감 정보가 포함되면 안 됨
