# 샘플 코드 5 — 알림 설정 및 발송

```java
@Service
public class NotificationService {

    @Autowired
    private NotificationSettingRepository settingRepository;

    @Autowired
    private NotificationLogRepository logRepository;

    @Autowired
    private EmailSender emailSender;

    @Autowired
    private SmsSender smsSender;

    @Autowired
    private PushSender pushSender;

    public void updateSettings(Long userId, Map<String, Boolean> settings) {
        NotificationSetting setting = settingRepository.findByUserId(userId);
        if (setting == null) {
            setting = new NotificationSetting();
            setting.setUserId(userId);
        }

        setting.setEmailEnabled(settings.getOrDefault("email", true));
        setting.setSmsEnabled(settings.getOrDefault("sms", false));
        setting.setPushEnabled(settings.getOrDefault("push", true));
        settingRepository.save(setting);
    }

    public void sendNotification(Long userId, String type, String message) {
        NotificationSetting setting = settingRepository.findByUserId(userId);

        if (setting.getEmailEnabled()) {
            String email = getUserEmail(userId);
            emailSender.send(email, type, message);
            logRepository.save(new NotificationLog(userId, "EMAIL", type, message, new Date()));
        }

        if (setting.getSmsEnabled()) {
            String phone = getUserPhone(userId);
            smsSender.send(phone, message);
            logRepository.save(new NotificationLog(userId, "SMS", type, message, new Date()));
        }

        if (setting.getPushEnabled()) {
            String deviceToken = getDeviceToken(userId);
            pushSender.send(deviceToken, type, message);
            logRepository.save(new NotificationLog(userId, "PUSH", type, message, new Date()));
        }
    }

    public void sendBulkNotification(String type, String message) {
        List<Long> allUserIds = settingRepository.findAllUserIds();
        for (Long userId : allUserIds) {
            sendNotification(userId, type, message);
        }
    }

    private String getUserEmail(Long userId) {
        return settingRepository.findByUserId(userId).getEmail();
    }

    private String getUserPhone(Long userId) {
        return settingRepository.findByUserId(userId).getPhone();
    }

    private String getDeviceToken(Long userId) {
        return settingRepository.findByUserId(userId).getDeviceToken();
    }
}
```

## 참고 맥락

- 암호화폐 거래소의 알림 서비스 (체결, 입출금, 보안 알림 등)
- 전체 사용자 대상 대량 알림 발송 기능 포함
- 보안 알림(비밀번호 변경, 로그인 시도 등)은 설정과 무관하게 발송 필수
- 알림 메시지에 민감정보(금액, 주소 등)가 포함될 수 있음
- 대량 발송 시 외부 서비스(이메일/SMS) 호출 횟수 및 비용 고려 필요
- 발송 실패 시 재시도 로직 필요
