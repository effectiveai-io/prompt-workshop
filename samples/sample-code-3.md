# 샘플 코드 3 — 주소 화이트리스트 관리

```java
@RestController
@RequestMapping("/api/whitelist")
@RequiredArgsConstructor
public class WhitelistController {

    private final WhitelistService whitelistService;
    private final AuthService authService;

    @PostMapping
    public ResponseEntity<WhitelistResponse> addAddress(
            @Valid @RequestBody AddressRequest request,
            @AuthenticationPrincipal UserPrincipal user) {

        WhitelistEntry entry = whitelistService.addAddress(
            user.getId(), request.getAddress(), request.getLabel(), request.getNetwork());

        return ResponseEntity.ok(WhitelistResponse.of(entry));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> removeAddress(
            @PathVariable Long id,
            @RequestParam String otpCode,
            @AuthenticationPrincipal UserPrincipal user) {

        authService.verifyOtp(user.getId(), otpCode);
        whitelistService.removeAddress(id, user.getId());

        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<WhitelistResponse>> getAddresses(
            @AuthenticationPrincipal UserPrincipal user) {
        return ResponseEntity.ok(whitelistService.getAddresses(user.getId()));
    }

    @PostMapping("/toggle")
    public ResponseEntity<Map<String, Boolean>> toggleWhitelist(
            @RequestParam boolean enabled,
            @AuthenticationPrincipal UserPrincipal user) {

        whitelistService.setWhitelistEnabled(user.getId(), enabled);
        return ResponseEntity.ok(Map.of("whitelistEnabled", enabled));
    }
}

@Service
@RequiredArgsConstructor
public class WhitelistService {

    private final WhitelistRepository whitelistRepository;
    private static final int MAX_ADDRESSES = 20;
    private static final long PENDING_HOURS = 24;

    public WhitelistEntry addAddress(Long userId, String address, String label, String network) {
        long count = whitelistRepository.countByUserId(userId);
        if (count >= MAX_ADDRESSES) {
            throw new BusinessException("최대 등록 가능 주소를 초과했습니다");
        }

        WhitelistEntry entry = WhitelistEntry.builder()
            .userId(userId)
            .address(address)
            .label(label)
            .network(network)
            .status(WhitelistStatus.PENDING)
            .activeAt(LocalDateTime.now().plusHours(PENDING_HOURS))
            .build();

        return whitelistRepository.save(entry);
    }

    public void removeAddress(Long id, Long userId) {
        WhitelistEntry entry = whitelistRepository.findById(id)
            .orElseThrow(() -> new BusinessException("주소를 찾을 수 없습니다"));

        if (!entry.getUserId().equals(userId)) {
            throw new UnauthorizedException("본인의 주소만 삭제할 수 있습니다");
        }

        whitelistRepository.delete(entry);
    }

    public List<WhitelistResponse> getAddresses(Long userId) {
        return whitelistRepository.findByUserId(userId).stream()
            .map(WhitelistResponse::of)
            .collect(Collectors.toList());
    }

    public void setWhitelistEnabled(Long userId, boolean enabled) {
        UserSetting setting = userSettingRepository.findByUserId(userId);
        setting.setWhitelistEnabled(enabled);
        userSettingRepository.save(setting);
    }

    public boolean isAddressAllowed(Long userId, String address) {
        UserSetting setting = userSettingRepository.findByUserId(userId);
        if (!setting.isWhitelistEnabled()) {
            return true;
        }

        return whitelistRepository.findByUserIdAndAddress(userId, address)
            .filter(entry -> entry.getStatus() == WhitelistStatus.ACTIVE)
            .isPresent();
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 출금 주소 화이트리스트 기능
- 보안이 최우선: 해킹 시 미등록 주소로 출금 차단
- 24시간 대기 기간은 보안을 위한 설계
- 주소 삭제 시 2FA(OTP) 필수
- 화이트리스트 ON→OFF 전환 시에도 24시간 대기 필요하지만 현재 미구현
- 최대 20개 주소 등록 가능
- 네트워크별 주소 형식 검증 필요 (BTC, ETH, TRX 등) — 현재 미구현
