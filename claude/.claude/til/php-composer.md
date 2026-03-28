# PHP Composer

> 학습일: 2026-02-11

## PHP 배포 방식이 다른 이유

PHP는 **인터프리터 언어** — 빌드(컴파일) 없이 소스코드 자체가 실행됨.

```
컴파일 언어 (Java, Go):    소스 → 빌드 → 바이너리 → 배포
프론트엔드 (Vue, React):   소스 → 빌드(번들링) → dist/ → 배포
PHP:                       소스 → 그대로 배포 → 서버에서 인터프리터 실행
```

## Composer = PHP 패키지 매니저

| PHP | Node.js | Python |
|-----|---------|--------|
| `composer` | `npm` | `pip` |
| `composer.json` | `package.json` | `requirements.txt` |
| `composer.lock` | `package-lock.json` | pip freeze |
| `vendor/` | `node_modules/` | `site-packages/` |
| `composer install` | `npm install` | `pip install -r` |

## composer install 동작

```
composer.json   ← 필요한 패키지 선언
composer.lock   ← 정확한 버전 고정
      ↓
composer install
      ↓
vendor/         ← 패키지 다운로드 + autoloader 생성
├── autoload.php
├── 패키지1/
└── 패키지2/
```

## 왜 서버에서 composer install을 실행하는가?

`vendor/`는 git이나 아카이브에 포함하지 않음:
1. **용량** — 수십~수백 MB
2. **환경 의존성** — 일부 패키지가 서버 OS/PHP 버전에 맞게 설치되어야 함
3. **관례** — `node_modules/`를 git에 안 넣는 것과 동일

## 실무 배포 흐름 (LSW 프로젝트 기준)

```
로컬: 소스코드 아카이브 (vendor/ 제외)
  ↓ SFTP 업로드
서버: 압축 해제 → composer install → vendor/ 생성 → 실행 가능
```
