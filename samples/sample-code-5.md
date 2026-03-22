# 샘플 코드 5 — 알림 설정 및 발송

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final NotificationSettingRepository settingRepository;
    private final NotificationLogRepository logRepository;
    private final EmailSender emailSender;
    private final SmsSender smsSender;
    private final PushSender pushSender;
    private final UserRepository userRepository;

    public void updateSettings(Long userId, NotificationSettingRequest request) {
        NotificationSetting setting = settingRepository.findByUserId(userId)
            .orElseGet(() -> NotificationSetting.createDefault(userId));

        setting.update(request.isEmailEnabled(), request.isSmsEnabled(), request.isPushEnabled());
        settingRepository.save(setting);
    }

    public void sendNotification(Long userId, NotificationType type, String message) {
        NotificationSetting setting = settingRepository.findByUserId(userId)
            .orElseGet(() -> NotificationSetting.createDefault(userId));

        User user = userRepository.findById(userId)
            .orElseThrow(() -> new BusinessException("사용자를 찾을 수 없습니다"));

        if (setting.isEmailEnabled()) {
            emailSender.send(user.getEmail(), type.getSubject(), message);
            saveLog(userId, NotificationChannel.EMAIL, type, message);
        }

        if (setting.isSmsEnabled() && user.getPhone() != null) {
            smsSender.send(user.getPhone(), message);
            saveLog(userId, NotificationChannel.SMS, type, message);
        }

        if (setting.isPushEnabled() && user.getDeviceToken() != null) {
            pushSender.send(user.getDeviceToken(), type.getSubject(), message);
            saveLog(userId, NotificationChannel.PUSH, type, message);
        }
    }

    public void sendBulkNotification(NotificationType type, String message) {
        List<NotificationSetting> allSettings = settingRepository.findAll();

        for (NotificationSetting setting : allSettings) {
            try {
                sendNotification(setting.getUserId(), type, message);
            } catch (Exception e) {
                log.error("알림 발송 실패 - userId: {}, type: {}, error: {}",
                    setting.getUserId(), type, e.getMessage());
            }
        }
    }

    @Async
    public void sendDelayedNotification(Long userId, NotificationType type,
                                         String message, long delayMinutes) {
        try {
            Thread.sleep(delayMinutes * 60 * 1000);
            sendNotification(userId, type, message);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.warn("지연 알림 중단 - userId: {}", userId);
        }
    }

    private void saveLog(Long userId, NotificationChannel channel,
                          NotificationType type, String message) {
        NotificationLog notificationLog = NotificationLog.builder()
            .userId(userId)
            .channel(channel)
            .type(type)
            .message(message)
            .build();
        logRepository.save(notificationLog);
    }

    @Transactional(readOnly = true)
    public List<NotificationLogDto> getLogs(Long userId, int limit) {
        return logRepository.findTopByUserIdOrderByCreatedAtDesc(userId, PageRequest.of(0, limit))
            .stream()
            .map(NotificationLogDto::from)
            .collect(Collectors.toList());
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 알림 서비스 (체결, 입출금, 보안 알림 등)
- 전체 사용자 대상 대량 알림 발송 기능 포함 (긴급 공지, 점검 안내 등)
- 보안 알림(비밀번호 변경, 로그인 시도, 출금 요청)은 설정과 무관하게 필수 발송이어야 함
- 알림 메시지에 민감정보(금액, 주소 등)가 포함될 수 있음
- 대량 발송 시 외부 서비스 호출 횟수 및 비용 고려 필요
- 지연 알림 기능은 가격 알림 등에 사용됨
