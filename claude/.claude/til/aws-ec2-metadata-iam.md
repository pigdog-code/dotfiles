# AWS EC2 메타데이터 서비스 & IAM Role

## EC2 Instance Metadata Service (IMDS)

### 메타데이터 주소

- **고정 IP**: `169.254.169.254` (링크-로컬 주소, RFC 3927)
- 전세계 모든 AWS 리전의 모든 EC2 인스턴스에서 동일한 주소 사용
- 외부 네트워크로 라우팅되지 않음 — 각 EC2 하이퍼바이저가 요청을 가로채서 해당 인스턴스의 정보만 반환
- AWS뿐 아니라 GCP, Azure도 동일한 IP를 메타데이터 서비스에 사용

### 주요 확인 명령어

| 항목                | 명령어                                                                                   | 설명                               |
|---------------------|------------------------------------------------------------------------------------------|--------------------------------------|
| **IAM Role 이름**   | `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/`              | Role 이름 반환 (없으면 404)          |
| **임시 자격증명**   | `curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/{Role이름}`    | AccessKeyId, Token, 만료 시간        |
| **Instance Profile**| `curl -s http://169.254.169.254/latest/meta-data/iam/info/`                              | InstanceProfileArn, RoleArn          |
| **Policy (권한)**   | 서버에서 직접 확인 불가 — AWS 콘솔 또는 `aws iam` CLI 필요 (별도 IAM 권한 필요)            | 실제 명령어 테스트로 간접 확인 가능   |

### 동작 원리

```
┌──────────────────┐     ┌──────────────────┐
│ EC2 (서울)        │     │ EC2 (오리건)      │
│                   │     │                   │
│ curl 169.254...  │     │ curl 169.254...  │
│   ↓               │     │   ↓               │
│ 하이퍼바이저가     │     │ 하이퍼바이저가     │
│ 가로채서 자기 정보 │     │ 가로채서 자기 정보 │
│ 반환 (충돌 없음)  │     │ 반환 (충돌 없음)  │
└──────────────────┘     └──────────────────┘
```

---

## IAM으로 EC2 → S3 연결하는 구조

### 3가지 구성 요소

```
EC2 Instance ──할당──▶ IAM Role ──포함──▶ IAM Policy ──허용──▶ S3 Bucket
 (서버)                (역할)              (정책)               (저장소)
```

| 구성 요소            | 역할                                   | 확인 방법                          |
|----------------------|----------------------------------------|-------------------------------------|
| **IAM Role**         | "이 서버는 이런 권한을 가진다" 정의     | 메타데이터 API로 확인 가능          |
| **IAM Policy**       | Role에 붙는 실제 권한 규칙              | AWS 콘솔에서만 확인 가능            |
| **Instance Profile** | EC2에 Role을 연결하는 브릿지            | 메타데이터 API로 확인 가능          |

### IAM Policy 예시

```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
  "Resource": "arn:aws:s3:::s3-uw2-he-analytics/*"
}
```

### 주요 S3 권한

| 권한              | 동작             | CLI 예시                           |
|-------------------|------------------|------------------------------------|
| `s3:ListBucket`   | 파일 목록 조회   | `aws s3 ls s3://bucket/path/`      |
| `s3:GetObject`    | 파일 다운로드    | `aws s3 cp s3://bucket/file .`     |
| `s3:PutObject`    | 파일 업로드      | `aws s3 cp file s3://bucket/`      |
| `s3:GetObjectAcl` | ACL 조회         | `aws s3api get-object-acl ...`     |
| `s3:PutObjectAcl` | ACL 설정         | `aws s3api put-object-acl ...`     |

### 권한 신청 시 필요한 정보

| 항목              | 내용                                     |
|-------------------|------------------------------------------|
| **EC2 정보**      | 어떤 서버에 Role을 붙일지 (서버 관리자)   |
| **S3 버킷 ARN**   | 어떤 버킷에 접근할지 (개발자)             |
| **필요한 권한**   | 어떤 동작을 허용할지 (개발자, 코드 기준)  |

### Policy 권한 테스트 방법 (간접 확인)

```bash
# ListBucket 테스트
aws s3 ls s3://버킷명/

# GetObject 테스트
aws s3 cp s3://버킷명/파일 /tmp/

# PutObject 테스트
echo "test" > /tmp/test.txt && aws s3 cp /tmp/test.txt s3://버킷명/test.txt
```

되면 권한 있음, `AccessDenied`면 없음.
