# 샘플 코드 3 — 주소 화이트리스트 관리

```java
@RestController
@RequestMapping("/api/whitelist")
public class WhitelistController {

    @Autowired
    private WhitelistRepository whitelistRepository;

    @Autowired
    private UserService userService;

    @PostMapping
    public ResponseEntity addAddress(@RequestBody AddressRequest request,
                                     @RequestHeader("Authorization") String token) {
        Long userId = userService.getUserId(token);

        WhitelistEntry entry = new WhitelistEntry();
        entry.setUserId(userId);
        entry.setAddress(request.getAddress());
        entry.setLabel(request.getLabel());
        entry.setNetwork(request.getNetwork());
        entry.setStatus("PENDING");
        entry.setCreatedAt(new Date());
        // 24시간 후 활성화
        entry.setActiveAt(new Date(System.currentTimeMillis() + 86400000));
        whitelistRepository.save(entry);

        return ResponseEntity.ok(Map.of("id", entry.getId(), "activeAt", entry.getActiveAt()));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity removeAddress(@PathVariable Long id,
                                        @RequestHeader("Authorization") String token,
                                        @RequestParam String otpCode) {
        Long userId = userService.getUserId(token);
        WhitelistEntry entry = whitelistRepository.findById(id).get();

        if (entry.getUserId() != userId) {
            return ResponseEntity.status(403).body("권한 없음");
        }

        // OTP 검증
        if (!userService.verifyOtp(userId, otpCode)) {
            return ResponseEntity.status(401).body("OTP 인증 실패");
        }

        whitelistRepository.delete(entry);
        return ResponseEntity.ok("삭제 완료");
    }

    @GetMapping
    public ResponseEntity getAddresses(@RequestHeader("Authorization") String token) {
        Long userId = userService.getUserId(token);
        List<WhitelistEntry> entries = whitelistRepository.findByUserId(userId);
        return ResponseEntity.ok(entries);
    }

    @PostMapping("/toggle")
    public ResponseEntity toggleWhitelist(@RequestHeader("Authorization") String token,
                                          @RequestParam boolean enabled) {
        Long userId = userService.getUserId(token);
        userService.setWhitelistEnabled(userId, enabled);
        return ResponseEntity.ok(Map.of("whitelistEnabled", enabled));
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 출금 주소 화이트리스트 기능
- 보안이 최우선: 해킹 시 미등록 주소로 출금 차단
- 24시간 대기 기간은 보안을 위한 설계
- 주소 삭제 시 2FA(OTP) 필수
- 화이트리스트 ON→OFF 전환 시에도 24시간 대기 필요
- 최대 20개 주소 등록 가능
- 네트워크별 주소 형식 검증 필요 (BTC, ETH, TRX 등)
