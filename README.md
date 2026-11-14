# WOOWANGTHON_TEAM_25_AI

우왕톤의 안내문 분석과 AI 영역 점수 계산을 담당하는 Python 패키지입니다. Django, HTTP 서버, 데이터베이스 없이 사용할 수 있습니다. 배포 이름은 `woowangthon-ai`, 가져오기 이름은 `woowang_ai`, 버전은 `0.1.0`입니다.

이 작업본은 GitHub `woowangthon/WOOWANGTHON_TEAM_25_AI`의 `main` 커밋 `45fa330bb267894a1ba13e6a1f5b9c4afc184c03`을 기준으로 합니다. 당시 저장소에는 제목만 있는 README 하나가 있었습니다. 기존 로컬 BE의 검증된 AI 구현을 이 패키지로 옮겼으며, GitHub에 push하거나 원격 파일을 변경하지 않았습니다.

## 설치와 테스트

Python 3.12 이상에서 이 디렉터리를 기준으로 실행합니다. 기존 BE 로컬 환경의 Python 3.12와 운영 Docker의 Python 3.13을 함께 지원합니다.

```bash
python -m venv .venv
# Windows PowerShell
.\.venv\Scripts\python.exe -m pip install -e ".[test]"
.\.venv\Scripts\python.exe -m unittest discover -s tests -t . -v
```

Linux에서는 `.\.venv\Scripts\python.exe`를 `.venv/bin/python`으로 바꿉니다. 테스트는 OpenAI SDK의 실제 파서와 `httpx.MockTransport`를 사용하며, 실제 모델이나 공공 API에 접속하지 않습니다. Django 설치나 설정도 필요하지 않습니다.

## 사용

```python
from woowang_ai.services import analyze_page_context
from woowang_ai.scoring import score_ai_analysis

text = "출금하려면 5만 원을 먼저 입금하세요."
analysis = analyze_page_context(page_context=text)
scored = score_ai_analysis(
    analysis,
    page_context=text,
    url="https://shop.example.com/withdrawal",
)
# scored.analysis: analyses.ai에 넣을 분석·근거·점수
# scored.score: 0~50 또는 확인 불가인 경우 None
```

`analyze_page_context(page_context="", page_observations=None)`는 안내문 또는 출처가 있는 관찰 기록을 분석합니다. 화면 수집은 FE, 입력 검증·인증·호출 제한·공공 API 조회·전체 점수 합산은 BE의 책임입니다. 이 패키지는 제공된 페이지 주소를 직접 방문하지 않습니다.

## 응답과 점수

- `status`: 분석 성공은 `completed`, 문구·키가 없으면 `skipped`, 시간 초과·API 오류·유효하지 않은 응답은 `error`입니다.
- `signals`: `type`, `evidence`, `reason`을 포함합니다. 관찰 기록을 사용하면 출처 ID·주소·종류·시각도 연결합니다. 원문에 없는 근거는 검증에서 제외하거나 오류로 처리합니다.
- `score_ai_analysis()`는 모델이 준 숫자를 신뢰하지 않고 검증한 근거로 AI 영역 점수를 계산합니다. 사업자정보 최대 50점과 AI 최대 50점인 기존 `business50_ai50_v3` 정책을 유지합니다. 사업자 점수와 최종 판정은 BE가 계산합니다.
- 직접 선입금 요구는 50점, 보상·수익 표시·출금 제한의 조합은 10점입니다. 공개 화면의 출금신청·충전신청 메뉴는 40점, 공지사항 없음·최근 1개월 공지 없음은 50점, 구매후기 없음은 40점입니다. 중복 점수를 반영한 원점수가 50점을 넘으면 AI 합계를 50점으로 제한합니다.
- 메뉴 존재·공지·후기 가점은 관찰 기록에 대한 규칙 점수이며 모델이 선입금 요구를 확인했다는 뜻은 아닙니다. 읽지 못한 화면은 미확인으로 남습니다. 점수는 사기 확률이 아닙니다.

## 모델과 입력 범위

`OPENAI_API_KEY`는 서버 환경변수에서만 읽습니다. `OPENAI_MODEL`로 모델을 선택하며 기본값은 `gpt-6-luna`입니다. 키를 FE·확장 프로그램·코드·로그에 넣지 마세요. `.env`를 자동으로 읽는 기능은 없으므로 실행 프로세스에 환경변수를 전달해야 합니다.

OpenAI Python SDK의 Responses API와 Structured Outputs를 사용합니다. 관련 안내문만 전달하고 URL·사업자정보를 모델 입력에 보태지 않습니다. 안내문은 최대 2,000자로 제한하며 원문 근거를 다시 검증합니다. 호출 시간 제한은 10초, SDK 재시도는 0회, 응답 저장은 `store=False`입니다. 출처·개인정보 제거와 화면의 가시성 확인은 기존 FE/BE 입력 경계를 유지해야 합니다.

## BE와 연결

형제 디렉터리 `WOOWANGTHON_TEAM_25_BE`가 이 패키지를 설치해 같은 서버 프로세스에서 호출합니다. 기존 BE의 `ai.services`, `ai.flow`, `ai.scoring`, `ai.site_checks`는 이 패키지의 실제 모듈을 가리키는 호환 경로입니다. `POST /api/v1/scan/`, `analyses.ai`, 시연용 기기 등록·인증·만료·호출 제한은 바뀌지 않습니다.

`site_checks.timezone.now()`는 표준 라이브러리 UTC 시계를 사용하는 작은 객체입니다. 테스트에서는 이 메서드를 patch해 날짜 경계를 검증할 수 있으며 Django 시간 설정을 요구하지 않습니다.

## 이번 작업에서 실행한 검증

별도로 만든 Python 3.12.14 가상환경에 이 패키지만 설치하고 Django가 설치되지 않았음을 확인한 뒤 위 `unittest` 명령으로 **79개 테스트를 모두 통과**했습니다. 모델 응답, 오류, 시간 초과는 모두 모의 처리했으며 실제 OpenAI 호출은 하지 않았습니다. wheel 빌드도 실행해 버전·Python 요구사항·운영 모듈 4개·Django 의존성 없음을 확인했습니다. FE/BE 연동과 운영 Docker 검증은 통합 작업본의 검사 결과를 함께 확인하세요.

