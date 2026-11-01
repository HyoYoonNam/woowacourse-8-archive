# 문서화 코드를 테스트에 두면 달라지는 것

책췍 도서 검색 API에 Spring REST Docs를 적용한 경험

남효윤 루드비코

## 1 REST Docs 도입 배경과 비교 범위

컨트롤러의 요청 처리 코드는 짧아도, API 설명과 응답 예시를 붙이면 읽어야 할 정보는 많아져요. 문서 설명도 필요하지만, 요청이 어느 서비스로 전달되는지 확인할 때는 그 설명이 먼저 눈에 들어오기도 해요. 문서화 정보를 테스트로 옮기면 컨트롤러에는 동작을 읽는 데 필요한 코드를 남길 수 있어요. 그런데 옮긴 뒤의 테스트와 빌드는 어떨까요?

책췍의 초기 도서 검색 API를 비교해 보니, REST Docs를 추가해도 컨트롤러는 그대로였고 성공 테스트는 38줄에서 46줄로 늘었어요. OpenAPI 명세를 연결한 뒤에는 60줄이 됐어요. 문서화 코드를 테스트에 두는 선택은 전체 코드를 줄여주지 않았어요. 늘어난 코드가 무엇을 확인해 주는지, 문서를 배포할 때 어떤 작업이 더 필요한지 살펴볼 필요가 있었어요.

저는 책 감상을 기록하고 공유하는 서비스 ‘책췍’에 REST Docs를 적용했어요. 이 글에서는 초기 도서 검색 API를 따라가며, 기존 MockMvc 테스트에 문서화를 연결하고 OpenAPI 명세를 만들어 Swagger UI로 제공하는 과정을 살펴볼 거예요. Spring MVC, MockMvc 테스트, Swagger를 이용한 문서화 경험이 있는 개발자를 대상으로 해요.

이 글에서는 세 가지를 확인할 거예요. 응답 필드의 문서화 정의를 빠뜨렸을 때 테스트가 어떻게 실패하는지, 설명의 뜻이 틀려도 왜 통과하는지, 문서 생성이 배포 빌드에 어떤 영향을 주는지예요. 앞의 두 가지는 별도 재현 예제로, 마지막은 책췍의 실제 배포 수정 기록으로 살펴볼게요.

### 도서 검색 API의 REST Docs 도입 이력

| 단계 | 커밋 | 변경 내용 |
| --- | --- | --- |
| 도입 직전 | [6466101](https://github.com/woowacourse-teams/2026-chaekchaek/commit/6466101) | 도서 검색 MockMvc 테스트 존재 |
| REST Docs 추가 | [b351d7d](https://github.com/woowacourse-teams/2026-chaekchaek/commit/b351d7d) | 기존 테스트에 요청과 응답 설명 추가 |
| UI 연결 완료 | [c3397bc](https://github.com/woowacourse-teams/2026-chaekchaek/commit/c3397bc) | OpenAPI 생성과 Swagger UI 제공 |

첫 두 커밋 사이에서는 BookControllerTest.java만 바뀌었어요. 검색 기능은 그대로 둔 채 문서화를 추가한 비교예요. 이 API는 처음부터 Swagger 애노테이션 없이 구현됐으므로, 다음 절의 애노테이션 코드는 같은 API를 설명하기 위해 재구성한 예시예요. 실제 제거 이력과 혼동하지 않도록 구분했어요.[^1][^2]

## 2 컨트롤러의 요청 처리 코드와 문서화 정보

도서 검색은 검색어와 페이지 번호를 받아 결과를 반환해요. 같은 메서드에 Swagger 애노테이션을 붙이면 다음과 같이 표현할 수 있어요. 아래는 문서화 정보가 놓이는 위치를 보여 주는 비교용 예시이며, import와 클래스 선언은 생략했어요.

```java
@Operation(summary = "도서 검색")
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "검색 성공"),
    @ApiResponse(responseCode = "400", description = "잘못된 요청")
})
@GetMapping("/api/v1/books")
public ResponseEntity<BookSearchResponse> search(
        @Parameter(description = "검색할 도서명")
        @RequestParam @NotBlank String query,
        @Parameter(description = "1부터 시작하는 페이지 번호")
        @RequestParam @Positive int page
) {
    return ResponseEntity.ok(bookSearchService.search(query, page));
}
```

이 예시는 메서드 본문까지 14줄이에요. 요청 경로 위에는 응답 설명이, 입력값 옆에는 파라미터 설명이 붙어 있어요. 아래 실제 코드와 비교하면 문서화 정보가 없는 메서드에서 요청 처리를 어떻게 읽을 수 있는지 보일 거예요. 이 비교에서는 문서화 정보의 유무에 따른 코드 구성을 볼 수 있어요.

책췍의 실제 검색 메서드 발췌[^1]

```java
@GetMapping("/api/v1/books")
public ResponseEntity<BookSearchResponse> search(
        @RequestParam @NotBlank String query,
        @RequestParam @Positive int page
) {
    return ResponseEntity.ok(bookSearchService.search(query, page));
}
```

실제 메서드는 같은 범위로 7줄이에요. 요청 매핑, 입력값과 검증 조건, 서비스 호출을 연달아 읽을 수 있어요. 여기서 @NotBlank와 @Positive는 요청을 검증하는 코드이므로 유지해요. 테스트로 옮기려는 대상은 이런 동작 자체가 아니라, 검색어의 뜻이나 응답 필드의 의미를 설명하는 문서화 정보예요.

Swagger라는 이름으로 묶어 부르던 도구도 구분할 필요가 있어요. springdoc-openapi는 Spring 코드와 애노테이션을 바탕으로 OpenAPI 명세를 만들고, Swagger UI는 그 명세를 화면에 보여 줘요. REST Docs를 사용하더라도 생성한 명세를 Swagger UI에 연결할 수 있어요. 책췍에서는 문서 생성을 테스트에 연결하고, 문서를 읽는 화면은 Swagger UI로 제공했어요.[^2][^5]

## 3 REST Docs 선택 근거와 웹 계층 테스트 구성

선택 기준은 문서 설명을 컨트롤러에 추가하지 않으면서, 응답이 달라졌을 때 설명도 확인하게 만드는 것이었어요. 비교 대상으로는 모의 API를 제공하는 Apidog와 요청 실행, 공유에 사용하는 Postman이 있어요. 이들을 문서와 요청을 별도로 관리하는 방식으로 쓴다면 서버 변경과 문서 갱신을 연결할 절차도 함께 정해야 해요.

우아한형제들의 2018년 적용 글은 코드와 별도로 관리하던 wiki 문서가 어긋난 경험에서 출발해요.[^8] 이 글에서도 문서 갱신을 코드 변경과 어떻게 연결할 수 있는지 살펴볼게요. Apidog와 Postman도 테스트와 자동화 구성에 따라 변경을 검증하는 방식이 달라질 수 있어요.

| 검토한 사용 방식 | 문서 설명 위치 | 함께 마련할 것 |
| --- | --- | --- |
| 애노테이션 기반 생성 | 컨트롤러와 DTO 등 | 실제 응답과 설명을 확인하는 테스트 |
| 별도 문서와 요청 관리 | Apidog나 Postman의 문서와 요청 | 서버 변경과 명세 검증을 연결하는 절차 |
| REST Docs 연결 | 요청을 실행하는 테스트 | 테스트 사례와 문서 생성 빌드 |

책췍에는 이미 도서 검색 컨트롤러를 확인하는 MockMvc 테스트가 있었어요. 그래서 별도 요청 사례를 처음부터 관리하기보다, 이 테스트가 주고받은 요청과 응답에 설명을 붙이는 방식을 선택했어요. 초기 도입 커밋에서 테스트 구성은 다음과 같아요.[^1]

```java
@WebMvcTest(BookController.class)
@AutoConfigureRestDocs
class BookControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private BookSearchService bookSearchService;
}
```

@WebMvcTest로 웹 계층을 구성하고, 검색 서비스는 Mockito 대역으로 바꿔요. 테스트는 HTTP 요청 매핑, 응답 직렬화와 필드 설명에 집중해요. 외부 도서 API의 응답이나 DB 조회까지 확인하는 테스트는 아니에요. 대역이 반환할 값을 정한 뒤 컨트롤러가 내보내는 결과를 확인하는 범위예요.

@AutoConfigureRestDocs는 MockMvc와 REST Docs를 연결해요. 이 프로젝트가 슬라이스 테스트를 선택한 것이지 REST Docs 자체가 이 방식만 요구하는 것은 아니에요. 전체 컨텍스트를 사용하는 테스트에 연결할 수도 있어요. 테스트 범위를 먼저 정하고 그 결과를 문서화해야, 생성된 문서가 무엇을 근거로 하는지도 분명해져요.[^4]

## 4 요청과 응답 문서화, 스니펫 생성

기존 검색 성공 테스트는 검색 서비스의 반환값을 준비하고, query=마션과 page=1로 요청한 뒤 상태 코드와 JSON 값을 확인해요. REST Docs를 도입하면서 이 검증을 지우지 않았어요. 마지막에 document()를 연결해 문서 조각인 스니펫을 생성했어요. 아래는 추가한 부분을 중심으로 줄인 코드예요.[^1]

```java
mockMvc.perform(get("/api/v1/books")
        .param("query", "마션").param("page", "1"))
    .andExpect(status().isOk())
    .andExpect(jsonPath("$.totalCount").value(11))
    .andExpect(jsonPath("$.nextPage").value(2))
    // 실제 테스트의 나머지 응답 값 검증은 생략
    .andDo(document("book-search",
        queryParameters(
            parameterWithName("query").description("검색할 도서명"),
            parameterWithName("page").description("1부터 시작하는 페이지 번호")
        ),
        responseFields(BOOK_SEARCH_RESPONSE_FIELDS)
    ));
```

queryParameters()에는 쿼리 파라미터 설명을, responseFields()에는 응답 본문의 필드 설명을 전달해요. BOOK\_SEARCH\_RESPONSE\_FIELDS는 다음과 같은 FieldDescriptor 배열이에요. 전체 응답을 엄격하게 문서화하려면 나머지 필드도 모두 기술해야 하므로, 이 발췌만으로 실제 응답 전체를 문서화할 수는 없어요.

```java
private static final FieldDescriptor[] BOOK_SEARCH_RESPONSE_FIELDS = {
    fieldWithPath("totalCount").type(JsonFieldType.NUMBER)
        .description("검색 결과의 전체 도서 수"),
    fieldWithPath("nextPage").type(JsonFieldType.NUMBER)
        .description("다음 페이지 번호. 마지막 페이지라면 null"),
    fieldWithPath("items").type(JsonFieldType.ARRAY)
        .description("검색된 도서 목록"),
    fieldWithPath("items[].title").type(JsonFieldType.STRING)
        .description("도서 제목")
    // 표지, 저자, 옮긴이, 출판일, ISBN, 분류, 출판사도 기술
};
```

items\[\].title은 배열 안의 도서 제목을 가리켜요. jsonPath()의 값 검증과 필드 문서화는 역할이 달라요. 앞의 검증은 준비한 값이 응답에 담겼는지 확인하고, 뒤의 문서화는 실행된 응답과 필드 목록을 대조해요. 설명 문구는 개발자가 직접 작성해요.

해당 커밋에서 backend 디렉터리로 이동해 ./gradlew test --tests '\*BookControllerTest'를 실행하면, 기본 출력 위치인 build/generated-snippets/book-search 아래에 요청, 응답, 쿼리 파라미터와 응답 필드 스니펫을 생성하는 구성이에요. 전체 클래스와 당시 Gradle 설정은 참고 자료의 고정 커밋에서 확인할 수 있어요. 예시의 패키지와 의존성은 Java 21, Spring Boot 4.1.0 기반 코드이므로 다른 버전에 옮길 때는 그대로 복사하기보다 버전에 맞춰 확인해야 해요.[^1][^2]

## 5 필드 문서화의 검증 범위와 한계

문서화 테스트가 통과하면 어디까지 믿을 수 있을까요? 이를 구분하려고 책췍의 초기 검색 응답 구조와 11개 필드 설명을 사용한 별도 재현 예제를 실행했어요. 고정 JSON을 반환하는 MockMvc 독립 컨트롤러에 문서화를 연결했으며, 운영 서버나 실제 검색 서비스를 호출한 결과는 아니에요. 기준 사례에서는 11개 필드를 모두 정의했어요. 두 번째 사례는 fieldWithPath("items\[\].publisher")로 시작하는 정의를 제외했고, 세 번째는 목록을 복원한 뒤 title의 .description("도서 제목")만 .description("도서 가격")으로 바꿨어요. 각 사례는 같은 고정 응답을 별도로 문서화했어요.

실행 환경은 Java 21, Spring Framework 7.0.8, Spring REST Docs 4.0.1이에요. 상태 코드 200을 확인한 뒤 queryParameters()와 responseFields()를 연결하고, 동일한 응답에 대해 필드 정의를 제거하거나 설명 문구를 바꿨을 때의 결과를 비교했어요.

| 실행한 변경 | 결과 | 확인한 범위 |
| --- | --- | --- |
| 11개 필드를 모두 문서화 | 통과 | 기준 응답을 문서화할 수 있음 |
| items\[\].publisher의 필드 정의 제거 | 실패 | 응답에 남은 미문서화 필드를 발견 |
| title 설명을 ‘도서 가격’으로 변경 | 통과 | 설명 문장의 의미는 검증하지 않음 |

publisher의 필드 정의를 제외한 두 번째 사례에서는 상태 코드 200 검증을 통과했지만, 문서를 생성하는 단계에서 아래 SnippetException이 발생했어요. 실패 메시지는 문서화되지 않은 필드가 어디인지 보여 줬어요.

```text
SnippetException:
The following parts of the payload were not documented:
{
  "items" : [ {
    "publisher" : "알에이치코리아(RHK)"
  } ]
}
```

세 번째 경우는 달랐어요. title의 설명을 ‘도서 가격’으로 바꿔도 문서 생성은 성공했어요. 필드 경로와 타입이 맞는다고 설명의 뜻까지 맞는 것은 아니었어요. 따라서 문서화 테스트의 통과 여부와 설명 문장의 정확성은 나눠 확인해야 해요. 실제 검색 결과나 도서 수 계산의 정확성도 고정 응답을 문서화한 이번 재현으로는 확인할 수 없어요.

설정도 검증 범위를 바꿔요. 기본 responseFields()는 문서에 없는 응답 필드나 optional로 표시하지 않은 필수 필드의 누락을 실패로 처리해요. relaxedResponseFields()를 사용하면 미문서화 필드가 있어도 허용해요. optional()은 해당 필드가 없는 경우를 허용하므로, ‘키가 항상 존재하며 값만 null일 수 있다’는 계약을 이것만으로 보장할 수는 없어요.[^6]

초기 도입 테스트의 nextPage는 숫자 2였고, 뒤의 OpenAPI 연결 커밋에서는 null 사례와 optional()이 추가됐어요. 문서에 ‘마지막 페이지라면 null’이라고 적는 것과 그 상황을 실제로 테스트하는 것은 별개예요. 마지막 페이지와 다음 페이지가 있는 경우를 각각 준비하고, 필요한 키 존재 여부와 값 조건도 별도로 검증하는 편이 좋아요.[^1][^2]

## 6 스니펫의 OpenAPI 변환과 Swagger UI 배포

스니펫 파일만 생성하면 클라이언트 개발자가 읽을 문서 화면은 아직 없어요. 책췍에서는 ePages의 restdocs-api-spec을 사용해 스니펫의 정보를 OpenAPI 3.0.1 명세로 모으고, 그 명세를 Swagger UI에 연결했어요. Spring REST Docs의 스니펫 생성과 OpenAPI 변환은 서로 다른 단계예요.[^2][^7]

[PR #23](https://github.com/woowacourse-teams/2026-chaekchaek/pull/23)에서는 document()의 import를 ePages의 MockMvcRestDocumentationWrapper로 바꾸고, resource()에 요약과 태그 등의 정보를 추가했어요. 응답 필드 배열을 재사용하면서 기존 responseFields()도 유지했어요. 아래는 해당 resource() 부분의 핵심만 발췌한 코드예요.

```java
import static com.epages.restdocs.apispec
    .MockMvcRestDocumentationWrapper.document;

// document()에 전달하는 스니펫 중 일부
resource(ResourceSnippetParameters.builder()
    .summary("도서 검색")
    .description("도서명과 페이지 번호로 도서를 검색한다")
    .tag("도서")
    // queryParameters 설정은 생략
    .responseFields(BOOK_SEARCH_RESPONSE_FIELDS)
    .build())
```

Gradle에는 com.epages.restdocs-api-spec 플러그인과 테스트용 restdocs-api-spec-mockmvc 의존성 0.20.1을 추가했어요. openapi3 설정에는 문서 이름과 출력 형식을 지정했고, 생성된 YAML은 build/api-spec/openapi3.yaml에 두었어요. 필요한 빌드 흐름은 다음 표처럼 연결돼요.

| 단계 | 입력 또는 역할 | 다음 단계에 넘기는 것 |
| --- | --- | --- |
| test | 문서화 테스트 실행 | generated-snippets |
| openapi3 | 스니펫 정보 집계 | openapi3.yaml |
| verifyOpenApi | 생성 명세의 경로와 응답 등 확인 | 검증을 통과한 명세 |
| stageOpenApiSpec | 명세를 정적 리소스 경로에 복사 | static/docs/openapi3.yaml |
| bootJar | UI와 명세를 실행 JAR에 포함 | 배포할 애플리케이션 |

이 구성에서는 명세를 생성하는 것뿐 아니라, 생성 결과에 검색 경로와 200, 400, 500, 502 응답이 들어 있는지도 verifyOpenApi로 확인했어요. 오류 응답이 application/problem+json과 ProblemDetail 스키마를 사용하도록 검사한 것도 프로젝트에서 추가한 검증이에요. REST Docs를 도입하면 저절로 생기는 검증은 아니에요.[^2]

당시 구성은 ./gradlew clean test verifyOpenApi로 테스트부터 생성 명세 검증까지 확인할 수 있어요. 이후 JAR에 명세를 포함해 배포하면 /docs/swagger-ui.html에서 읽게 돼요. 독자가 따라 적용할 때도 테스트 성공, 명세 생성, 최종 배포물 포함 여부를 나눠 확인해야 해요. 문서 화면이 열리는지만 보지 말고, 수정한 필드와 예시가 명세에 반영됐는지도 함께 확인해 보세요.

## 7 문서화 코드 증가와 배포 시 스니펫 생성 비용

도입 전후에 컨트롤러는 그대로였어요. 변한 곳은 테스트였고, 그 테스트는 더 길어졌어요. 아래는 검색 성공 테스트의 메서드 선언부터 닫는 중괄호까지 빈 줄과 주석을 포함해 센 결과예요. 컨트롤러 파일 전체의 줄 수도 함께 비교했어요.

| 측정 대상 | 도입 전 | REST Docs 추가 | OpenAPI 연결 |
| --- | --- | --- | --- |
| 검색 성공 테스트 메서드 | 38줄 | 46줄 | 60줄 |
| 검색 컨트롤러 파일 전체 | 26줄 | 26줄 | 26줄 |

테스트 메서드 밖의 응답 필드 배열, import와 빌드 설정은 첫 행에 포함하지 않았어요. 따라서 이 표는 전체 유지보수 비용의 합계가 아니에요. REST Docs 추가 커밋에서는 테스트 파일이 134줄에서 176줄로 늘었어요. 새 문서화 정보를 작성하는 비용이 생겼다는 사실을 보여 주는 수치예요.[^1][^2]

늘어난 설명을 계속 관리할 수 있을까요? 응답 필드를 추가하면 테스트 데이터, 값 검증, 필드 설명을 함께 살펴야 해요. 테스트 응답에 새 필드가 실제로 포함됐는데 설명이 없다면 엄격한 필드 문서화가 실패해요. 설명을 작성할 일은 남지만, 빠뜨렸다는 사실을 테스트 결과에서 확인할 수 있어요. 반면 뜻이 틀린 설명은 앞의 재현처럼 리뷰가 필요해요.

### 스니펫 누락에 따른 CD 빌드 실패와 수정

문서화의 영향은 테스트 파일에서 끝나지 않았어요. [PR #62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62)에는 CD에서 bootJar를 만들다가 openapi3 단계가 실패한 기록이 있어요. 기존 명령은 CI에서 테스트를 했다는 이유로 CD에서는 테스트를 생략했어요. 하지만 CD 빌드 환경에는 명세를 생성할 스니펫이 없었어요. 테스트 성공 여부와 문서 생성에 필요한 파일의 존재는 서로 다른 조건이었어요.[^3]

```bash
# 변경 전
./gradlew clean bootJar -x test

# PR #62에서 변경한 명령
./gradlew clean bootJar --no-daemon
```

[PR #62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62)에서는 CD에서도 테스트를 실행해 스니펫을 만들도록 바꿨어요. 테스트가 검증뿐 아니라 문서 생성의 입력을 만드는 단계이기도 했기 때문이에요. CI에서 한 번 통과했어도 그때 만든 파일을 CD로 전달하지 않으면 다음 빌드에서 사용할 수 없어요. 이 수정으로 문서 생성에 필요한 입력을 마련했지만, PR에는 CI와 CD의 중복 테스트 실행이 단점으로 남았어요.

후속 개선에서는 무엇을 다음 단계로 넘길지 결정해야 해요. CI에서 검증한 JAR를 그대로 배포하면 CD에서 다시 문서를 만들 필요가 없고, CD에서 새로 빌드한다면 같은 커밋에서 만든 문서 산출물을 전달하거나 다시 생성해야 해요. 이는 아직 적용했다고 말할 수 없는 설계 선택지예요. 테스트를 생략할지보다, 배포할 코드와 문서가 같은 변경을 반영하는지부터 확인해야 해요.

## 8 REST Docs 도입 판단과 적용 점검

도서 검색 API 하나에서 확인한 변화는 분명했어요. 컨트롤러는 그대로였고, 문서 설명은 테스트에 추가됐어요. 그 덕분에 설명이 빠진 응답 필드를 문서 생성 과정에서 발견할 수 있었어요. 대신 설명의 뜻은 사람이 검토해야 했고, 배포 빌드에는 스니펫을 마련하는 단계가 필요해졌어요. 문서화를 테스트에 둔다는 것은 이 작업들까지 함께 관리하는 선택이었어요.

이미 웹 계층 테스트를 작성하고 있으며 응답 변경 때 문서 갱신을 놓치는 문제가 있다면, 대표 API 하나에 연결해 볼 만해요. 반대로 테스트 기반이 거의 없거나 구현 전 명세 합의가 중심인 팀이라면, 테스트를 새로 마련하는 비용과 문서가 필요한 시점을 먼저 살펴야 해요. REST Docs가 모든 팀에서 가장 싼 문서화 방식인 것은 아니에요.

처음 적용할 때는 응답 구조가 단순하고 정상 응답과 오류 응답을 모두 준비할 수 있는 API를 고르면 좋아요. 기존 값 검증을 유지한 채 필드 설명을 추가하고, 응답에 있는 필드의 문서화 정의 하나를 일부러 제외해 실패하는지 확인해 보세요. 이어서 생성된 명세와 최종 문서 화면에 같은 요청과 응답이 나타나는지 확인하면, 코드 작성부터 배포까지 필요한 작업이 드러나요.

우아한형제들의 후속 글에서는 API가 늘어난 뒤 빌드 속도와 문서 탐색의 불편을 개선해요.[^9] 책췍의 다음 개선도 테스트 실행 비용과 문서를 읽는 사람의 사용성을 함께 살펴볼 필요가 있어요.

[^1]: [책췍 도서 검색 REST Docs 도입 전후 비교](https://github.com/woowacourse-teams/2026-chaekchaek/compare/64661016078a8973291d4c4d824cae9fd5c9b15d...b351d7d7e59e4490f60c5f5b0163d21ad5eddf8b) 테스트 원문과 추가한 11개 응답 필드 설명. 도입 직후 커밋 b351d7d.

[^2]: [책췍 REST Docs 기반 Swagger 문서 제공 PR 23](https://github.com/woowacourse-teams/2026-chaekchaek/pull/23) 본문 기준 리비전 c3397bc. backend/build.gradle과 BookControllerTest.java 참고.

[^3]: [책췍 CD에서 테스트 스킵 제거 PR 62](https://github.com/woowacourse-teams/2026-chaekchaek/pull/62) 문서 생성 입력이 없어 배포 빌드가 실패한 사례와 수정 명령.

[^4]: [Spring Creating API Documentation with Restdocs](https://spring.io/guides/gs/testing-restdocs/) 테스트 구성과 스니펫 생성 방식. 사용 중인 버전의 문서와 함께 확인.

[^5]: [springdoc openapi 공식 문서](https://springdoc.org/) Spring 코드와 애노테이션을 바탕으로 OpenAPI 명세를 만드는 방식.

[^6]: [Spring REST Docs Request and Response Payloads](https://docs.spring.io/spring-restdocs/reference/documenting-your-api/request-response-payloads.html) 응답 필드 경로, optional과 relaxed 모드의 검증 범위.

[^7]: [ePages restdocs api spec](https://github.com/ePages-de/restdocs-api-spec) REST Docs 스니펫을 OpenAPI로 변환하는 확장 도구. 프로젝트 사용 버전은 0.20.1.

[^8]: [Spring Rest Docs 적용, 우아한형제들 기술블로그](https://techblog.woowahan.com/2597/) 2018-12-28

[^9]: [Spring REST Docs에 날개를… (feat: Popup), 우아한형제들 기술블로그](https://techblog.woowahan.com/2678/) 2020-05-13
