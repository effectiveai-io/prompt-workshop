# 샘플 코드 4 — 거래 내역 조회 API

```java
@RestController
@RequestMapping("/api/transactions")
public class TransactionController {

    @Autowired
    private TransactionRepository transactionRepository;

    @Autowired
    private UserService userService;

    @GetMapping
    public ResponseEntity getTransactions(
            @RequestHeader("Authorization") String token,
            @RequestParam(required = false) String type,
            @RequestParam(required = false) String startDate,
            @RequestParam(required = false) String endDate,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "50") int size) {

        Long userId = userService.getUserId(token);

        List<Transaction> transactions;
        if (type != null && startDate != null && endDate != null) {
            transactions = transactionRepository.findByUserIdAndTypeAndCreatedAtBetween(
                userId, type,
                new SimpleDateFormat("yyyy-MM-dd").parse(startDate),
                new SimpleDateFormat("yyyy-MM-dd").parse(endDate)
            );
        } else if (type != null) {
            transactions = transactionRepository.findByUserIdAndType(userId, type);
        } else {
            transactions = transactionRepository.findByUserId(userId);
        }

        // 페이징 처리
        int fromIndex = page * size;
        int toIndex = Math.min(fromIndex + size, transactions.size());
        List<Transaction> paged = transactions.subList(fromIndex, toIndex);

        Map<String, Object> response = new HashMap<>();
        response.put("data", paged);
        response.put("total", transactions.size());
        response.put("page", page);
        response.put("size", size);

        return ResponseEntity.ok(response);
    }

    @GetMapping("/export")
    public ResponseEntity exportCsv(
            @RequestHeader("Authorization") String token,
            @RequestParam String startDate,
            @RequestParam String endDate) {

        Long userId = userService.getUserId(token);
        List<Transaction> transactions = transactionRepository
            .findByUserIdAndCreatedAtBetween(
                userId,
                new SimpleDateFormat("yyyy-MM-dd").parse(startDate),
                new SimpleDateFormat("yyyy-MM-dd").parse(endDate)
            );

        StringBuilder csv = new StringBuilder();
        csv.append("날짜,종류,금액,상태,주소\n");
        for (Transaction tx : transactions) {
            csv.append(tx.getCreatedAt() + ","
                + tx.getType() + ","
                + tx.getAmount() + ","
                + tx.getStatus() + ","
                + tx.getAddress() + "\n");
        }

        return ResponseEntity.ok()
            .header("Content-Type", "text/csv")
            .header("Content-Disposition", "attachment; filename=transactions.csv")
            .body(csv.toString());
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 거래 내역 조회/내보내기 API
- 사용자별로 수만 건의 거래 내역이 존재할 수 있음
- CSV 내보내기는 세무 신고용으로 사용됨
- 날짜 파싱 에러, 페이징 범위 초과, 메모리 이슈 등 고려 필요
- 개인정보(주소)가 CSV에 포함되므로 보안 검토 필요
