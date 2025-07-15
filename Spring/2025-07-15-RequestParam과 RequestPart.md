# @RequestParam vs @RequestPart

## @RequestParam 의 사전적 정의

HTTP 요청의 파라미터(query parameter 또는 form parameter)를 메서드 파라미터에 바인딩하는 애노테이션

### 특징

- 단순한 키-값 쌍의 파라미터를 처리
- URL 쿼리 스트링 (`?name=value`)또는 폼 데이터를 받음
- 기본 타입 변환만 지원 (String, int, boolean 등)

### 예시

```
GET /api/users?name=홍길동&age=25
POST /api/upload (form-data: title=제목, file=파일)
```

## @RequestPart 의 사전적 정의

`multipart/form-data`요청에서 각각의 파트(part)를 메서드 파라미터에 바인딩하는 애노테이션

### 특징

- multipart 요청의 개별 파트를 처리
- 각 파트의 Content-Type을 인식하여 적절한 변환 수행
- 복잡한 객체나 JSON 데이터도 처리 가능

### 예시

```
POST /api/upload
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"
Content-Type: application/pdf
[파일 데이터]

--boundary
Content-Disposition: form-data; name="metadata"
Content-Type: application/json
{"title": "문서제목", "author": "작성자"}
```

## 주요 차이점

| 구분                  | @RequestParam        | @RequestPart                |
| --------------------- | -------------------- | --------------------------- |
| **Content-Type 처리** | 무시                 | 각 파트의 Content-Type 인식 |
| **데이터 변환**       | 단순 타입 변환       | HttpMessageConverter 사용   |
| **복잡한 객체**       | 지원 제한적          | JSON 등 복잡한 객체 지원    |
| **사용 시나리오**     | 단순한 파일 + 텍스트 | 파일 + 복잡한 메타데이터    |

## 선택가이드

### @RequestParam

- 단순한 파일 업로드
- 텍스트 파라미터와 파일이 함께 있는 경우
- 기본적인 form 데이터 처리

### RequestPart

- 파일과 함께 JSON 객체를 전송하는 경우
- 각 파트의 Content-Type이 다른 경우
- RESTful API에서 구조화된 데이터와 파일을 함께 처리하는 경우

단순한 경우라면 @RequestParam이, 복잡한 데이터 구조가 필요하다면 @RequestPart가 더 적합합니다.
