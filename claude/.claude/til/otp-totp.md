# OTP / TOTP 동작 원리

## OTP란?
- One-Time Password, 한 번만 사용 가능한 일회용 비밀번호
- 주로 2FA(Two-Factor Authentication)의 두 번째 인증 수단으로 사용

## 두 가지 방식

### HOTP (HMAC-based OTP) - RFC 4226
- **카운터 기반**: 사용할 때마다 카운터 증가
- `OTP = Truncate(HMAC-SHA1(Secret, Counter)) mod 10^6`
- 다음 OTP 생성 전까지 무기한 유효 → 상대적으로 보안 낮음
- 사용 예: YubiKey HOTP

### TOTP (Time-based OTP) - RFC 6238
- **시간 기반**: 30초마다 새 OTP 생성
- `OTP = Truncate(HMAC-SHA1(Secret, floor(UnixTime / 30))) mod 10^6`
- 30초 후 자동 만료 → 더 높은 보안
- 사용 예: Google Authenticator, Microsoft Authenticator, Authy

## TOTP 핵심 원리

```
서버와 클라이언트(OTP 앱)가 같은 Secret Key를 보유하고,
같은 알고리즘으로 독립적으로 같은 값을 계산한다.
→ 양쪽 간에 OTP 값을 주고받는 통신은 없음
```

### Secret Key 공유 시점
- **최초 1회**: QR코드 스캔으로 Secret Key 전달 (유일한 공유 시점)
- **이후**: 서로 통신 없이 독립적으로 OTP 계산

### QR코드 URI 형식
```
otpauth://totp/{label}?secret={base32_secret}&issuer={issuer}&digits=6&period=30&algorithm=SHA1
```

## Dynamic Truncation (6자리 추출 과정)

```
1. HMAC-SHA1(Secret, Counter) → 20바이트 해시
2. 마지막 바이트의 하위 4비트 → offset
3. offset 위치에서 4바이트 추출
4. 최상위 비트 제거 (& 0x7FFFFFFF)
5. mod 10^6 → 6자리 OTP
```

## Unix Time은 전 세계 공통

- 1970-01-01 00:00:00 UTC부터 경과한 초(seconds)
- 시간대(timezone)와 무관하게 전 세계 동일한 값
- 그래서 서버가 어느 나라에 있든 동일한 TOTP 계산 가능

```
서울 (UTC+9)  2026-02-12 20:00 → Unix: 1771063200
런던 (UTC+0)  2026-02-12 11:00 → Unix: 1771063200  (동일)
뉴욕 (UTC-5)  2026-02-12 06:00 → Unix: 1771063200  (동일)
```

## 시간 오차 허용

- 서버는 보통 현재 스텝 ±1~2 스텝 허용 (약 ±30~60초)
- 서버: NTP로 정확한 시간 유지
- 스마트폰: OS가 네트워크 시간 자동 동기화
- Google Authenticator에 "시간 동기화" 기능이 있는 이유

## OTP 전달 방식 비교

| 방식 | 보안성 | 비고 |
|------|--------|------|
| 앱 기반 (TOTP) | 높음 | 오프라인 동작, 탈취 어려움 |
| SMS | 중간 | SIM 스와핑, SS7 공격 취약 |
| 이메일 | 중간 | 이메일 계정 탈취 시 위험 |
| 하드웨어 토큰 | 매우 높음 | 물리적 소유 필요 |

## 보안 고려사항

1. **Secret Key 보호**: 서버 DB에 암호화 저장
2. **Rate Limiting**: 브루트포스 방지 (6자리 = 100만 가지)
3. **재사용 방지**: 같은 타임스텝 내 동일 OTP 재사용 차단
4. **백업 코드**: Authenticator 앱 분실 대비 일회용 백업 코드 제공
