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

        if (wallet.getBalance().compareTo(amount) >= 0) {
            BigDecimal newBalance = wallet.getBalance().subtract(amount);
            wallet.updateBalance(newBalance);
        } else {
            throw new InsufficientBalanceException("잔액이 부족합니다");
        }

        BigDecimal fee = calculateFee(amount);

        Transaction tx = Transaction.builder()
            .userId(userId)
            .amount(amount)
            .address(address)
            .type(TransactionType.WITHDRAW)
            .status(TransactionStatus.PENDING)
            .fee(fee)
            .netAmount(amount.subtract(fee))
            .build();
        transactionRepository.save(tx);

        notificationService.sendWithdrawNotification(userId, tx);

        log.info("출금 요청 처리 완료 - userId: {}, amount: {}, fee: {}, txId: {}",
            userId, amount, fee, tx.getId());

        return WithdrawResponse.of(tx);
    }

    private BigDecimal calculateFee(BigDecimal amount) {
        BigDecimal feeRate = new BigDecimal("0.001");
        BigDecimal fee = amount.multiply(feeRate);
        BigDecimal minFee = new BigDecimal("0.0001");
        return fee.compareTo(minFee) < 0 ? minFee : fee;
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
- 규제: 일일 출금 한도 확인 필요, 출금 주소 화이트리스트 확인 필요
- 수수료 정책: 출금 금액의 0.1%, 최소 수수료 0.0001 BTC
- 알림에 민감 정보가 포함되면 안 됨
