# heat-man.github.io
CAT 프로젝트를 수정해줘.

프로젝트 목적은 Windows EVTX/XML, 특히 Sysmon 로그를 분석하고, 분석 결과를 LM Studio에 전달하여 침해사고 분석 보고서를 생성하는 것이다.

현재까지 확인된 문제와 원하는 수정사항은 다음과 같다.

## 1. 대용량 XML/Sysmon 파싱 Timeout 개선

현재 약 90MB 크기의 Sysmon XML을 분석할 때 다음 오류가 발생한다.

`XML parsing exceeded 60 seconds`

현재 XML 파싱 제한은 `CAT_XML_PARSE_TIMEOUT_SECONDS` 환경변수로 제어되고 기본값이 약 60초로 설정되어 있다.

다음과 같이 수정해라.

* 기본 XML 파싱 timeout을 60초에서 300초로 변경
* 환경변수 `CAT_XML_PARSE_TIMEOUT_SECONDS`를 통한 override 기능은 유지
* 최대 허용값 1800초는 유지
* 대용량 XML을 메모리에 한 번에 모두 읽지 말고 현재 스트리밍 파싱 구조가 있다면 유지
* timeout 발생 시 파일 크기, 파싱된 이벤트 수, 경과시간 등을 로그에 가능하면 남겨라
* 기존 EVTX/XML 분석 기능을 깨뜨리지 마라

권장 기본값:

`CAT_XML_PARSE_TIMEOUT_SECONDS=300`

## 2. LM Studio Context 초과 문제를 고려한 입력 제한

90MB Sysmon 분석 시 LM Studio 요청이 다음과 같이 실패했다.

`request (73514 tokens) exceeds the available context size (64000 tokens)`

LM Studio의 context length 자체는 별도로 128k 수준까지 올릴 예정이지만, CAT도 무제한으로 이벤트를 LM에 보내면 안 된다.

현재 프로젝트에 존재하는 아래 제한값들을 검토해라.

* DEFAULT_LM_MAX_INPUT_CHARS
* DEFAULT_LM_MAX_FIELD_CHARS
* MAX_LM_FINDINGS
* MAX_LM_EVIDENCE_PER_FINDING
* MAX_LM_SUSPICIOUS_EVENTS
* MAX_LM_SCENARIO_CANDIDATES
* MAX_LM_TIMELINE_EVENTS

다음 방향으로 개선해라.

* LM 입력은 전체 로그를 그대로 전달하지 않는다.
* CAT의 로컬 분석 결과 중 의미 있는 이벤트와 evidence를 우선 전달한다.
* 동일하거나 반복되는 Sysmon 이벤트는 가능한 범위에서 축약한다.
* 대량의 정상 이벤트가 LM prompt를 잠식하지 않게 한다.
* 입력 제한으로 일부 데이터가 제외된 경우 LM에게 "전체 이벤트 중 일부만 제공되었다"는 사실을 명확하게 알려라.
* 보고서에 evidence limitation으로도 표시할 수 있게 한다.
* token 계산 라이브러리를 새로 강제 의존성으로 추가하지 않아도 된다.
* 문자 수 기반의 보수적 제한을 사용해도 된다.
* 64k context에서도 최대한 안전하게 동작하고, 128k에서는 여유 있게 동작하도록 설계한다.

## 3. LM Studio timeout 설정 유지 및 개선

이전에 LM Studio 응답이 900초 제한을 초과한 적이 있다.

LM timeout은 환경변수로 조정할 수 있도록 유지하고, 기본 동작을 명확하게 정리해라.

권장:

* `CAT_LM_TIMEOUT_SECONDS`
* 기본값은 현재 값을 검토하여 필요하면 900초 유지
* 최대값은 충분히 크게 허용
* timeout 발생 시 단순 "timeout"만 출력하지 말고 다음 정보를 가능한 범위에서 로그에 남겨라.

  * 요청 모델
  * 입력 크기
  * 요청 시작 후 경과시간
  * 사용한 endpoint

단, API key나 민감정보는 로그에 출력하지 마라.

## 4. 가장 중요한 수정: LM Studio 보고서 스키마 강제 제거

현재 가장 불편한 문제는 LM Studio 응답을 CAT의 고정 JSON Schema에 맞추도록 강제한다는 것이다.

예를 들어 다음 오류가 발생했다.

`LM Studio 구조화 보고서 필수 섹션이 없습니다: analysis_scope, attack_scenarios, evidence_limitations, executive_summary, no_scenario_reason, recommendations`

내가 원하는 동작은 이것이 아니다.

LM Studio가 반드시 아래와 같은 고정 필드를 반환할 필요는 없다.

* analysis_scope
* attack_scenarios
* evidence_limitations
* executive_summary
* no_scenario_reason
* recommendations
* 기타 기존 required sections

LM Studio가 주어진 증거를 자유롭게 분석하도록 하고 싶다.

즉 다음 구조로 변경해라.

기존:

CAT 분석 결과
→ LM Studio
→ json_schema 강제
→ 필수 필드 검증
→ 누락되면 RuntimeError

변경:

CAT 분석 결과
→ LM Studio
→ 자유로운 분석 응답
→ JSON이면 가능한 범위에서 파싱
→ Markdown이면 그대로 사용
→ 일반 텍스트여도 그대로 사용
→ 보고서 화면에 정상 출력

## 5. CAT_LM_STRICT_VALIDATION 동작 변경

`CAT_LM_STRICT_VALIDATION` 환경변수를 두 가지 운영 모드로 사용해라.

### strict=true

기존의 구조화 보고서 기능을 유지한다.

* response_format=json_schema 사용 가능
* required sections 검증
* schema validation 수행
* 형식이 잘못되면 명확한 validation error 발생

즉 기존 기능을 완전히 삭제하지 마라.

### strict=false

이 모드를 기본 운영 모드로 사용하고 싶다.

strict=false일 때는:

* `response_format=json_schema`를 LM Studio 요청에 넣지 마라.
* LM Studio에 특정 JSON Schema를 강제하지 마라.
* Markdown 응답을 허용해라.
* 일반 텍스트 응답을 허용해라.
* JSON 응답도 허용해라.
* JSON이 반환되면 기존 구조화 보고서 렌더링을 활용할 수 있으면 활용한다.
* JSON 구조가 일부 누락되었다고 RuntimeError를 발생시키지 마라.
* JSON 파싱에 실패해도 오류 처리하지 말고 LM 원문을 보고서로 사용한다.
* 필수 section 누락으로 분석 전체가 실패해서는 안 된다.

기본값은 가능하면:

`CAT_LM_STRICT_VALIDATION=false`

로 변경해라.

## 6. LM 프롬프트 수정

현재 프롬프트에서 다음과 같은 문구가 있다면 strict=false일 때 제거해라.

예:

`응답은 API의 response_format JSON schema를 정확히 따르는 JSON 객체 하나만 반환하며 Markdown이나 code fence를 추가하지 않는다.`

또는 `_structured_output_instructions()`처럼 고정 JSON 출력을 요구하는 프롬프트.

strict=false일 때의 프롬프트는 다음 취지를 반영하도록 수정해라.

"다음 CAT 분석 결과를 바탕으로 Windows 침해사고 조사 보고서를 한국어로 작성하라. 정해진 JSON 형식이나 고정된 보고서 섹션을 반드시 따를 필요는 없다. 제공된 증거에서 의미 있는 내용을 우선적으로 분석하고, 근거가 부족한 내용은 추정하지 말고 가설 또는 추가 확인 필요 사항으로 구분한다. Markdown 형식의 자유로운 보고서를 반환할 수 있다."

그리고 다음 원칙을 포함해라.

* 증거 없는 악성 판단 금지
* Event ID, 시간, Process, CommandLine, IP, Domain 등 실제 evidence 중심
* 공격 시나리오는 근거가 있을 때만 제시
* 근거가 부족하면 "확인되지 않음"이라고 명시
* 정상 가능성이 있는 이벤트는 정상 가능성도 설명
* 대량 로그가 축약되어 전달된 경우 그 한계를 명시
* 한국어 보고서 작성

## 7. 자유 형식 응답 처리

LM 응답 처리 함수, 예를 들어 `_generate_lm_report`, `_recover_lm_report`, `validate_structured_report` 등의 흐름을 검토해라.

strict=false에서는 다음 우선순위를 사용해라.

1. LM 응답이 비어 있지 않은지 확인
2. JSON으로 파싱 가능하면 JSON으로 처리 시도
3. 기존 CAT 구조와 호환되는 필드가 있으면 기존 Markdown renderer 활용 가능
4. 일부 필드가 없더라도 누락 필드를 이유로 실패시키지 않는다
5. JSON이 아니면 Markdown 또는 텍스트 원문을 그대로 report_markdown으로 반환
6. Markdown code fence가 있어도 보고서 생성 실패로 처리하지 않는다
7. LM 응답이 완전히 비어 있는 경우에만 오류 처리

기존에 다음과 같이 표시하는 부분이 있다면:

`LM Studio 원문 (검증되지 않음)`

자유 모드에서는 반드시 이런 경고 제목을 강제로 붙일 필요는 없다.

대신 필요하면 상태 metadata에 다음과 같이 기록해라.

* structured_report_validated
* structured_report_recovered
* unstructured_report_used
* validation_warnings

프론트엔드가 이 값을 사용하지 않더라도 API 호환성을 해치지 않는 범위에서 유지해도 된다.

## 8. LM Studio URL/Origin 관련 기존 보안 기능 유지

현재 LM Studio는 VM에서 Host의 VMware 인터페이스를 통해 접근할 수 있다.

예:

`http://192.168.100.1:1234`

현재 존재하는 다음 설정은 유지한다.

* LM_STUDIO_URL
* CAT_ALLOW_CUSTOM_LM_URL
* CAT_LM_ALLOWED_ORIGINS

`CAT_LM_ALLOWED_ORIGINS`는 정확한 origin만 허용하는 현재 보안 정책을 유지해라.

예:

정상:

`http://192.168.100.1:1234`

잘못된 예:

`http://192.168.100.1:1234/v1`
`http://192.168.100.1:1234/v1/chat/completions`

이 보안 검증은 이번 작업에서 제거하지 마라.

## 9. 기존 기능 호환성

다음 기능은 반드시 유지해라.

* EVTX 업로드
* XML 업로드
* Sysmon 분석
* 분석 시간 범위 제한
* Rule 기반 분석
* LM Studio 분석
* multipart 업로드 제한
* 동시 분석 lock
* HTTP header timeout
* response timeout
* 파일 크기 제한
* Origin 검증
* Custom LM endpoint 검증
* 기존 API endpoint
* `/api/health`
* `/api/analyze`
* static UI

기존 테스트가 있다면 최대한 통과하도록 수정해라.

## 10. 테스트 추가

아래 테스트를 추가하거나 기존 테스트에 반영해라.

### Test 1

strict=false에서 LM이 정상 JSON 전체 schema를 반환

기대:
정상 보고서 출력

### Test 2

strict=false에서 LM이 일부 필드만 있는 JSON 반환

예:

```json
{
  "summary": "PowerShell 의심 행위 확인",
  "findings": ["EncodedCommand 실행"]
}
```

기대:
RuntimeError 없이 보고서 출력

### Test 3

strict=false에서 LM이 Markdown 반환

예:

```markdown
# Sysmon 분석 결과

## 주요 행위
PowerShell에서 의심스러운 명령이 실행되었습니다.
```

기대:
그대로 report_markdown에 출력

### Test 4

strict=false에서 일반 텍스트 반환

기대:
그대로 보고서 출력

### Test 5

strict=true에서 required section이 누락된 JSON 반환

기대:
기존처럼 validation error 발생

### Test 6

LM 응답이 빈 문자열

기대:
명확한 오류 발생

### Test 7

대량 analysis 객체 입력

기대:
LM에 전달되는 입력이 제한값 이하로 축약되며 정상 요청

## 11. 코드 품질

수정 시 다음을 지켜라.

* 임시 hack보다 기존 구조를 활용
* 같은 기능을 중복 구현하지 말 것
* strict/relaxed 분기를 명확하게 할 것
* 함수가 지나치게 커지면 helper로 분리
* 기존 변수명과 스타일 최대한 유지
* 사용자 입력이나 로그 내용을 과도하게 잘라내더라도 핵심 evidence는 우선 보존
* security validation을 단순히 우회하지 말 것

## 12. 수정 후 결과 보고

작업 완료 후 다음을 정리해서 알려줘.

1. 수정한 파일 목록
2. 각 파일에서 수정한 핵심 내용
3. strict=true / false 동작 차이
4. 변경된 환경변수 및 기본값
5. 대용량 Sysmon 처리 개선 내용
6. LM 입력 축약 방식
7. LM Studio 자유 응답 처리 방식
8. 추가한 테스트
9. 테스트 실행 결과
10. 기존 기능에 미칠 수 있는 영향

가능하면 실제 코드를 수정한 후 `pytest` 또는 프로젝트에 존재하는 테스트를 실행해서 회귀 오류가 없는지 확인해라.

가장 중요한 목표는 다음 두 가지다.

첫째, 대용량 Sysmon/EVTX 로그 때문에 CAT 전체 분석이 쉽게 실패하지 않아야 한다.

둘째, `CAT_LM_STRICT_VALIDATION=false`에서는 LM Studio가 고정된 보고서 양식에 맞추지 않더라도 실제 분석 내용을 정상적으로 받아서 사용자에게 보고서로 보여줘야 한다.
