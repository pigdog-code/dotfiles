# DB Foreign Key CASCADE

## Foreign Key (FK)란?

테이블 간 관계를 정의하는 제약 조건. 자식 테이블의 컬럼이 부모 테이블의 PK를 참조한다.

```sql
-- content.id_category는 categories.id_category를 참조
ALTER TABLE content
ADD FOREIGN KEY (id_category) REFERENCES categories(id_category);
```

## CASCADE란?

부모 레코드가 삭제/수정될 때, **자식 레코드를 자동으로 함께 처리**하는 옵션.

### ON DELETE CASCADE

```sql
ALTER TABLE content
ADD FOREIGN KEY (id_category) REFERENCES categories(id_category) ON DELETE CASCADE;
```

- 카테고리 30 삭제 → 카테고리 30에 속한 콘텐츠 **자동 삭제**
- DB가 알아서 정리해줌

### ON DELETE 옵션 종류

| 옵션            | 동작                                         |
|-----------------|----------------------------------------------|
| `CASCADE`       | 부모 삭제 시 자식도 자동 삭제                |
| `SET NULL`      | 부모 삭제 시 자식의 FK 컬럼을 NULL로 변경    |
| `RESTRICT`      | 자식이 존재하면 부모 삭제 **거부** (에러)     |
| `NO ACTION`     | RESTRICT와 동일 (MySQL/MariaDB 기준)         |
| (미지정)        | DB 엔진 기본값 적용 (보통 RESTRICT)          |

## CASCADE 있으면 vs 없으면

```
[CASCADE ON DELETE 있음]
카테고리 30 삭제 → 콘텐츠 10건 자동 삭제 ✅

[CASCADE 없음 — FK만 있거나 FK도 없음]
카테고리 30 삭제 → 콘텐츠 10건 DB에 남음
                    → id_category = 30인데 카테고리 30은 없음
                    → "고아 레코드" (orphaned records) ⚠️
```

## CASCADE가 없으면 어떻게 해야 하나?

**애플리케이션 코드에서 직접 관리**해야 한다:

```php
// 카테고리 삭제 전에 콘텐츠 존재 여부 확인
$contents = $model->getContentsByCategory($categoryId);
if (count($contents) > 0) {
    throw new Exception('콘텐츠가 존재하는 카테고리는 삭제할 수 없습니다.');
}
$model->deleteCategory($categoryId);
```

또는:

```php
// 콘텐츠 삭제 시 관련 데이터 수동 정리
$model->deleteContentProperty($contentId);  // 자식1 삭제
$model->deletePosterProps($contentId);       // 자식2 삭제
$model->deleteContent($contentId);           // 부모 삭제
```

## 실무 패턴

### CASCADE를 쓰는 경우
- 부모 없이 자식이 의미 없을 때 (주문 삭제 → 주문항목 삭제)
- 관계가 단순하고 명확할 때

### CASCADE를 안 쓰는 경우
- 실수로 대량 삭제 방지 (안전 우선)
- 삭제 전 검증 로직이 필요할 때
- 레거시 시스템에서 FK 자체를 안 쓰는 경우 (코드로만 관계 관리)

### 주의사항
- CASCADE는 **연쇄 삭제**가 가능 → 부모 → 자식 → 손자 전부 삭제될 수 있음
- 의도치 않은 대량 삭제 위험이 있어서, 운영 DB에서는 신중하게 설정

## 비교: 다른 언어/프레임워크에서

| 프레임워크     | CASCADE 설정 방법                         |
|----------------|-------------------------------------------|
| MySQL/MariaDB  | `ON DELETE CASCADE` (DDL)                 |
| Laravel (PHP)  | `$table->foreign('id')->onDelete('cascade')` |
| Django (Python) | `on_delete=models.CASCADE`               |
| TypeORM (TS)   | `@ManyToOne(() => Parent, { onDelete: 'CASCADE' })` |
| Prisma (TS)    | `@relation(onDelete: Cascade)`           |
