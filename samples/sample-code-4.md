# 샘플 코드 4 — 거래 내역 조회 API

```java
@RestController
@RequestMapping("/api/transactions")
@RequiredArgsConstructor
public class TransactionController {

    private final TransactionService transactionService;

    @GetMapping
    public ResponseEntity<PageResponse<TransactionDto>> getTransactions(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam(required = false) String type,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
            @RequestParam(required = false) @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {

        TransactionSearchCondition condition = TransactionSearchCondition.builder()
            .userId(user.getId())
            .type(type != null ? TransactionType.valueOf(type) : null)
            .startDate(startDate)
            .endDate(endDate)
            .build();

        Page<TransactionDto> result = transactionService.search(condition,
            PageRequest.of(page, size, Sort.by("createdAt").descending()));

        return ResponseEntity.ok(PageResponse.of(result));
    }

    @GetMapping("/export")
    public ResponseEntity<byte[]> exportCsv(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate) {

        List<Transaction> transactions = transactionService.findAllByPeriod(
            user.getId(), startDate, endDate);

        StringBuilder csv = new StringBuilder();
        csv.append("날짜,종류,금액,수수료,상태,메모\n");
        for (Transaction tx : transactions) {
            csv.append(String.format("%s,%s,%s,%s,%s,%s\n",
                tx.getCreatedAt().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME),
                tx.getType(),
                tx.getAmount(),
                tx.getFee(),
                tx.getStatus(),
                tx.getMemo() != null ? tx.getMemo().replace(",", " ") : ""));
        }

        byte[] bytes = csv.toString().getBytes(StandardCharsets.UTF_8);
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_TYPE, "text/csv; charset=UTF-8")
            .header(HttpHeaders.CONTENT_DISPOSITION,
                "attachment; filename=transactions_" + startDate + "_" + endDate + ".csv")
            .body(bytes);
    }

    @GetMapping("/summary")
    public ResponseEntity<TransactionSummary> getSummary(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate startDate,
            @RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd") LocalDate endDate) {

        return ResponseEntity.ok(transactionService.getSummary(user.getId(), startDate, endDate));
    }
}

@Service
@RequiredArgsConstructor
public class TransactionService {

    private final TransactionRepository transactionRepository;

    public Page<TransactionDto> search(TransactionSearchCondition condition, Pageable pageable) {
        return transactionRepository.searchByCondition(condition, pageable)
            .map(TransactionDto::from);
    }

    public List<Transaction> findAllByPeriod(Long userId, LocalDate start, LocalDate end) {
        return transactionRepository.findByUserIdAndCreatedAtBetween(
            userId,
            start.atStartOfDay(),
            end.atTime(LocalTime.MAX));
    }

    public TransactionSummary getSummary(Long userId, LocalDate start, LocalDate end) {
        List<Transaction> transactions = findAllByPeriod(userId, start, end);

        BigDecimal totalDeposit = transactions.stream()
            .filter(tx -> tx.getType() == TransactionType.DEPOSIT)
            .map(Transaction::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal totalWithdraw = transactions.stream()
            .filter(tx -> tx.getType() == TransactionType.WITHDRAW)
            .map(Transaction::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal totalFee = transactions.stream()
            .map(Transaction::getFee)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return TransactionSummary.builder()
            .totalDeposit(totalDeposit)
            .totalWithdraw(totalWithdraw)
            .totalFee(totalFee)
            .transactionCount(transactions.size())
            .build();
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 거래 내역 조회/내보내기/요약 API
- 사용자별로 수만~수십만 건의 거래 내역이 존재할 수 있음
- CSV 내보내기는 세무 신고용으로 사용됨 — 금액 정확성이 매우 중요
- 기간 제한 없이 요청 가능 (1년치 요청 시 메모리/성능 이슈)
- summary API는 대시보드에서 실시간으로 호출됨
