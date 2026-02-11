# 브라우저 랜더링 / API 관련 Interview 질문 정리

## 1. 브라우저 렌더링 관련 질문 (핵심 원리)

Q. 주소창에 google.com을 입력하면 일어나는 과정을 설명해주세요.

- 포인트: DNS 조회 → TCP/TLS 핸드쉐이크 → HTTP 요청/응답 → 렌더링 과정 순으로 답변

A.

1. URL 파싱: 브라우저는 입력된 주소를 확인하고 HSTS 목록을 참조해 HTTP를 HTTPS로 전환하거나 유효성을 검사합니다.

2. DNS 조회: 브라우저/OS 캐시를 먼저 확인하고, 없다면 DNS 서버를 통해 도메인명에 해당하는 IP 주소를 찾아냅니다.

3. 연결 수립: 해당 IP의 서버와 TCP 3-Way Handshake를 통해 연결을 설정하고, HTTPS라면 TLS Handshake를 추가로 거쳐 보안 연결을 마칩니다.

4. 요청 및 응답: 브라우저가 HTTP GET 요청을 보내면, 서버(웹 서버/WAS)는 해당 리소스를 찾아 응답합니다.

5. 브라우저 렌더링: 브라우저는 받은 HTML/CSS를 파싱해 DOM/CSSOM 트리를 만들고, 이를 합친 Render Tree를 기반으로 Layout(배치)과 Paint(그리기) 과정을 거쳐 화면을 출력합니다. 중간에 JS를 만나면 파싱을 멈추고 엔진이 코드를 실행합니다..

---

Q. Critical Rendering Path(주요 렌더링 경로)의 단계를 설명해주세요.

- 포인트: DOM 트리 생성 → CSSOM 트리 생성 → Render Tree 생성 → Layout → Paint 단계를 정확히 짚어야 함

Q. Reflow(Layout)와 Repaint의 차이는 무엇이며, 언제 발생하나요?

- 포인트: 기하학적 수치(너비, 높이)가 변하면 Reflow, 색상 등 외관만 변하면 Repaint가 발생함을 인지하고, 성능을 위해 Reflow를 줄이는 방법

Q. `<script>` 태그를 HTML 하단에 두는 이유와 async, defer 속성의 차이를 설명해주세요.

- 포인트: 파싱 중단(Blocking) 현상을 이해하고 있는지 확인##

## 2. API 및 네트워크 관련 질문 (통신과 보안)

Q. HTTP와 HTTPS의 차이점은 무엇인가요?

- 포인트: SSL/TLS 암호화와 보안 인증 과정을 이해

Q. RESTful API란 무엇이며, 설계 시 가장 중요하게 생각하는 원칙은?

- 포인트: 자원(Resource), 행위(Verb), 표현(Representation)의 분리를 설명

Q. CORS(Cross-Origin Resource Sharing)란 무엇이며, 어떻게 해결하나요?

- 포인트: 보안 정책상의 이유와 함께, 프록시 설정이나 서버 측 헤더(Access-Control-Allow-Origin) 수정을 언급

Q. GET과 POST 메소드의 차이를 설명해주세요.

- 포인트: 데이터 전송 방식, 캐싱 여부, 멱등성(Idempotency) 개념을 섞어 답변하면 더 좋음

Q. Cookie, LocalStorage, SessionStorage의 차이점은 무엇인가요?

- 포인트: 만료 기한, 저장 용량, 서버 전송 여부 등에 따른 적절한 사용처를 구분

## 3. 심화 및 실무 연계 질문

Q. 브라우저 렌더링 성능을 최적화하기 위해 프론트엔드 개발자가 할 수 있는 노력은?

- (답변 팁: CSS 하단 배치 지양, 이미지 레이지 로딩, 폰트 최적화, 애니메이션 시 transform 사용 등)

Q. API 호출 횟수를 줄이기 위한 전략이 있다면?

- (답변 팁: 디바운싱(Debouncing), 쓰로틀링(Throttling), HTTP 캐싱, SWR/React Query 같은 라이브러리 활용)
