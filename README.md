# Excel Validator for Spring Boot

Spring Boot 기반의 엑셀 업로드 파싱 및 유효성 검증 공통 유틸리티입니다.
검증 로직을 공통화하고, 업무별로 검증 규칙을 유연하게 주입할 수 있도록 설계되었습니다.

---

## 기술 스택

| 항목 | 버전 |
|------|------|
| Java | 21 |
| Spring Boot | 3.4.4 |
| Apache POI | 5.2.3 |
| Gradle | - |
| Lombok | - |
| JUnit 5 | - |

---

## 주요 기능

- `@ExcelColumn` 어노테이션 기반의 자동 컬럼 매핑 및 타입 변환
- Excel 데이터의 행 단위 유효성 검증
- 업무별 검증 로직을 **람다** 또는 **Validator 구현체**로 주입 가능
- 유효성 실패 시 예외 발생 및 전역 처리
- 에러 정보는 JSON 형태로 반환 (행 번호, 필드명, 에러 메시지)
- 재사용 가능한 제네릭 구조

---

## 프로젝트 구조

<details>
<summary>디렉토리 트리 보기</summary>

```
src/
├── main/java/com/example/excelvalidator/
│   ├── ExcelValidatorApplication.java
│   ├── core/
│   │   ├── exceptionHandler/
│   │   │   └── GlobalExceptionHandler.java     # 전역 예외 처리기
│   │   └── exceptions/
│   │       └── ExcelValidateException.java      # 유효성 검증 커스텀 예외
│   ├── excel/
│   │   ├── parser/
│   │   │   ├── ExcelParser.java                 # Apache POI 기반 엑셀 파싱 유틸
│   │   │   └── annotations/
│   │   │       └── ExcelColumn.java             # 컬럼 인덱스 매핑 어노테이션
│   │   └── validator/
│   │       ├── ExcelValidator.java              # 검증 실행 유틸 (@UtilityClass)
│   │       ├── ExcelRowValidator.java           # 행 단위 검증 함수형 인터페이스
│   │       └── vo/
│   │           └── ExcelErrorVo.java            # 에러 정보 VO (record)
│   └── sample/                                  # 사용 예시 구현체
│       ├── controller/SampleController.java
│       ├── dto/
│       │   ├── SampleExcelDto.java
│       │   └── SampleExcelReqDto.java
│       ├── excelValidator/SampleExcelValidator.java
│       └── service/SampleService.java
├── test/
│   ├── java/.../sample/service/SampleServiceTest.java
│   └── resources/
│       ├── sample-excel-request.json
│       └── sample-upload.xlsx
└── main/resources/
    └── application.properties
```

</details>

---

## 핵심 컴포넌트

### `@ExcelColumn`

DTO 필드에 Excel 컬럼 인덱스(0-based)를 매핑하는 어노테이션입니다.

```java
@Getter
public class SampleExcelDto {

    @ExcelColumn(index = 0)
    private String username;

    @ExcelColumn(index = 1)
    private String email;

    @ExcelColumn(index = 2)
    private String birthdate;
}
```

### `ExcelParser`

Apache POI를 사용해 엑셀 파일을 파싱하고 DTO 리스트로 변환합니다.
첫 번째 시트의 1행(헤더)을 건너뛰고 데이터를 읽습니다.

| 셀 타입 | 변환 결과 |
|---------|---------|
| STRING | 문자열 (trim) |
| NUMERIC | int / double / LocalDate |
| BOOLEAN | boolean |
| FORMULA | 수식 평가 결과 |
| BLANK | null |

```java
List<SampleExcelDto> list = ExcelParser.parse(multipartFile, SampleExcelDto.class);
```

### `ExcelValidator`

파싱된 행 리스트를 순회하며 검증 로직을 실행하고, 오류가 있으면 `ExcelValidateException`을 던집니다.

```java
ExcelValidator.validateAll(list, validator);
```

### `ExcelRowValidator<T>`

행 단위 검증 로직을 정의하는 함수형 인터페이스입니다.
람다 또는 구현체 방식 모두 사용 가능합니다.

```java
@FunctionalInterface
public interface ExcelRowValidator<T> {
    List<ExcelErrorVo> validate(T row, int rowNumber);
}
```

### `ExcelErrorVo`

에러 정보를 담는 불변 record 클래스입니다.

```java
public record ExcelErrorVo(int rowNumber, String fieldName, String errorMessage) {}
```

---

## 사용 방법

### 1. DTO 정의

```java
@Getter
public class SampleExcelDto {

    @ExcelColumn(index = 0)
    private String username;

    @ExcelColumn(index = 1)
    private String email;

    @ExcelColumn(index = 2)
    private String birthdate;
}
```

### 2-A. 람다 방식

```java
ExcelValidator.validateAll(excelList, (row, rowNum) -> {
    List<ExcelErrorVo> errors = new ArrayList<>();

    if (StringUtils.isBlank(row.getUsername())) {
        errors.add(new ExcelErrorVo(rowNum, "username", "아이디는 필수입니다."));
    }
    if (StringUtils.isBlank(row.getEmail()) || !row.getEmail().matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
        errors.add(new ExcelErrorVo(rowNum, "email", "이메일 형식이 올바르지 않습니다."));
    }

    return errors;
});
```

### 2-B. 구현체 방식

```java
public class SampleExcelValidator implements ExcelRowValidator<SampleExcelDto> {

    @Override
    public List<ExcelErrorVo> validate(SampleExcelDto row, int rowNumber) {
        List<ExcelErrorVo> errors = new ArrayList<>();

        if (StringUtils.isBlank(row.getUsername())) {
            errors.add(new ExcelErrorVo(rowNumber, "username", "아이디는 필수입니다."));
        }
        if (StringUtils.isBlank(row.getEmail()) || !row.getEmail().matches("^[A-Za-z0-9+_.-]+@(.+)$")) {
            errors.add(new ExcelErrorVo(rowNumber, "email", "이메일 형식이 올바르지 않습니다."));
        }
        if (StringUtils.isBlank(row.getBirthdate()) || !row.getBirthdate().matches("^\\d{4}-\\d{2}-\\d{2}$")) {
            errors.add(new ExcelErrorVo(rowNumber, "birthdate", "날짜 형식은 yyyy-MM-dd 입니다."));
        }

        return errors;
    }
}
```

```java
ExcelValidator.validateAll(excelList, new SampleExcelValidator());
```

---

## 예외 및 에러 처리 구조

```
ExcelValidator.validateAll()
    → 오류 존재 시 ExcelValidateException 발생
        → GlobalExceptionHandler (@ControllerAdvice)
            → HTTP 400 Bad Request + List<ExcelErrorVo> JSON 반환
```

### 에러 응답 예시

```json
[
  {
    "rowNumber": 3,
    "fieldName": "username",
    "errorMessage": "아이디는 필수입니다."
  },
  {
    "rowNumber": 4,
    "fieldName": "email",
    "errorMessage": "이메일 형식이 올바르지 않습니다."
  }
]
```

---

## API 엔드포인트 (Sample)

| Method | URL | 설명 |
|--------|-----|------|
| POST | `/api/v1/samples/excel/impl` | 구현체 방식 검증 |
| POST | `/api/v1/samples/excel/lamda` | 람다 방식 검증 |
| POST | `/api/v1/samples/excel/parse` | 엑셀 파일 파싱 후 데이터 반환 |

---

## 테스트 실행

```bash
./gradlew test
```

테스트 리소스 위치:

```
src/test/resources/sample-excel-request.json   # JSON 기반 테스트 데이터
src/test/resources/sample-upload.xlsx          # 엑셀 파일 업로드 테스트용
```

---

## 작성자

김정현 ([@DevK-Jung](https://github.com/DevK-Jung))

---

## 라이선스

MIT License
