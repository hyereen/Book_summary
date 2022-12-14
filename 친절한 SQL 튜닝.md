7# 친절한 SQL 튜닝
- 개발자를 위한 SQL 튜닝 입문서
- DB 프로그래밍에 어느정도 경험이 쌓였는데도 성능 문제를 스스로 해결하지 못해 늘 고민하는 사람을 위한 책

## 1. SQL 처리과정과 I/O
### 1.1 SQL 파싱과 최적화
- SQL은 기본적으로 구조적, 집합적, 선언적인 질의 언어
- 원하는 결과를 구조적, 집합적으로 선언하지만, 그 결과집합을 만드는 과정은 절차적 -> 프로시저 필요
- SQL 옵티마이저: 프로시저를 만들어내는 DBMS 내부 엔진
- SQL 최적화: DBMS 내부에서 프로시저를 작성하고 컴파일해서 실행 가능한 상태로 만드는 전 과정

#### 1.1.2 SQL 최적화
- SQL 실행하기 전 최적화 과정
  1. SQL 파싱
     - 사용자로부터 SQL을 전달받으면 가장 먼저 SQL 파서가 파싱을 진행
     - 파싱 트리 생성: SQL 문을 이루는 개별 구성 요소를 분석해서 파싱트리 생성
     - Syntax 체크: 문법적 오류가 없는지 확인
     - Semantic 체크: 의미상 오류가 없는지 확인
  2. SQL 최적화
     - 옵티마이저: 미리 수집한 시스템 및 오브젝트 통계정보를 바탕으로 다양한 실행 경로를 생성해서 비교한 후 가장 효율적인 하나를 선택
     - 데이터베이스 성능을 결정하는 가장 핵심적인 엔진
  3. 로우 소스 생성
     - 로우 소스 생성기: SQL 옵티마이저가 선택한 실행 경로를 실제 실행 가능한 코드 또는 프로시저 형태로 포맷팅하는 단계

#### 1.1.3 SQL 옵티마이저
- SQL 옵티마이저: 사용자가 원하는 작업을 가장 효율적으로 수행할 수 있는 최적의 데이터액세스 경로를 선택해주는 DBMS의 핵심 엔진
- 옵티마이저의 최적화 단계 
  1. 사용자로부터 전달받은 쿼리를 수행하는 데 후보군이 될만한 실행계획들을 찾아냄
  2. 데이터 딕셔너리에 미리 수집해 둔 오브젝트 통계 및 시스템 통계정보를 이용해 각 실행계획의 예상비용 산정
  3. 최저 비용을 나타내는 실행계획 선택

### 1.2 SQL 공유 및 재사용
#### 1.2.1 소프트 파싱 vs 하드 파싱
- 라이브러리 캐시
  - SQL 파싱, 최적화, 로우 소스 생성 과정을 거쳐 생성한 내부 프로시저를 반복 재사용할 수 있도록 캐싱해 두는 메모리 공간
  - SGA 구성요소 중 하나
  - ㄴ SGA(System Global Area): 서버 프로세스와 백그라운드 프로세스가 공통으로 액세스하는 데이터와 제어구조를 캐싱하는 메모리 공간
  - 필요한 이유
    - 데이터베이스에서 이루어지는 처리과정은 대부분 IO작업에 집중되지만, 하드 파싱은 CPU를 많이 소비하는 작업
    - 이렇게 생성한 내부 프로시저를 한 번만 사용하고 버린다면 비효율이므로 라이브러리 캐시가 필요
- 소프트 파싱 Soft parsing: 사용자가 SQL문을 전달하면 DBMS는 SQL을 파싱한 후 해당 SQL이 라이브러리 캐시에 존재해서 곧바로 실행 단계로 넘어가는 것
- 하드 파싱 Hard parsing: 사용자가 SQL문을 전달하면 DBMS는 SQL을 파싱한 후 해당 SQL이 라이브러리 캐시에 존재하지 않아 찾는 데 실패해 최적화 및 로우 소스 생성 단계까지 모두 거치는 것

#### 1.2.2 바인드 변수의 중요성
- 이름없는 SQL 문제
  - SQL은 따로 이름이 없음 
  - 전체 SQL 텍스트가 이름 역할을 함
  - 처음 실행할 때 최적화 과정을 거쳐 동적으로 생성한 내부 프로시저를 라이브러리 캐시에 적재함으로써 여러 사용자가 공유하면서 재사용함
  - 캐시 공간이 부족하면 버려졌다가 다음에 다시 실행할 때 똑같은 최적화 과정을 거쳐 캐시에 적재됨
  - SQL 자체가 이름이기 때문에 텍스트 중 작은 부분이라도 수정되면 그 순간 다른 객체가 새로 탄생하는 구조
  - DBMS에서 수행되는 SQL이 모두 완성된 것은 아니므로 모두 저장하면 많은 공간이 필요하고 SQL을 찾는 속도도 느려짐 -> 오라클 같은 DBMS가 SQL 영구 저장을 하지 않음
  - 공유가능 SQL
    - 프로시저를 여러 개 생성하는 것이 아니라 파라미터가 있는 프로시저 하나를 공유하면서 재사용 하는 것이 낫다
    - 파라미터 Driven 방식으로 SQL을 작성하는 방법이 제공됨 -> 바인드 변수
    - 하드파싱은 최초에 한번만 일어나고 캐싱된 SQL을 여러 사람이 공유하면서 재사용하게 됨

### 1.3 데이터 저장 구조 및 I/O 메커니즘
#### 1.3.1 SQL이 느린 이유
- SQL이 느린 이유는 디스크 I/O 때문

#### 1.3.2 데이터베이스 저장 구조
- 데이터를 저장하려면 먼저 테이블스페이스를 생성해야 함
- 테이블스페이스
  - 세그먼트를 담는 콘테이너
  - 여러 개의 데이터파일(디스크 상의 물리적인 OS파일)로 구성됨
- 테이블 스페이스 > 세그먼트 > 익스텐트 > 블록 > 로우 
- 세그먼트: 데이터 저장공간이 필요한 오트벡트(테이블, 인덱스, 파티션, LOB 등)
  - 테이블, 인덱스처럼 데이터 저장공간이 필요한 오브젝트, 여러 익스텐트로 구성됨
- 테이블, 인덱스를 생성할 때 데이터를 어떤 테이블 스페이스에 저장할 지를 지정
- 파티션 구조가 아니라면 테이블도, 인덱스도 각각 하나의 세그먼트가 됨
- 테이블 또는 인덱스가 파티션 구조라면, 각 파티션이 하나의 세그먼트가 된다.
- LOB 컬럼은 그 자체가 하나의 세그먼트를 구성하므로 자신이 속한 테이블과 다른 별도의 공간에 값을 저장 
  - ㄴ LOB (Large Object): 그래픽이미지나 음성, 바이너리 데이터, 사이즈가 큰 텍스트 데이터를 다루는 데이터 타입
- 익스텐트: 공간을 확장하는 단위, 연속된 블록 집합
  - 공간을 확장하는 단위
  - 테이블이나 인덱스에 데이터를 입력하다가 공간이 부족해지면 해당 오브젝트가 속한 테이블 스페이스로부터 익스텐트를 추가로 할당받음
  - 연속된 블록들의 집합이기도 함
  - 한 익스텐트도 하나의 테이블이 독점 -> 한 익스텐트에 담긴 블록은 모두 같은 테이블 블록
- 블록: 데이터를 읽고 쓰는 단위
  - 익스텐트 단위로 공간을 확장하지만, 사용자가 입력한 레코드를 실제로 저장하는 공간은 데이터 블록
  - 한 블록은 하나의 테이블이 독점
  - 한 블록에 저장된 레코드는 모두 같은 테이블 레코드
- 데이터 파일: 디스크 상의 물리적인 OS 파일

#### 1.3.4 시퀀셜 액세스 vs 랜덤 액세스
- 테이블 또는 인덱스를 액세스하는(=읽는) 방식
- 시퀀셜 액세스 sequential access
  - 논리적 또는 물리적으로 연결된 순서에 따라 차례대로 블록을 읽는 방식
  - 인덱스 리프 블록은 앞 뒤를 가리키는 주소값을 통해 논리적으로 서로 연결돼 있음
  - 이 주소 값에 따라 앞 또는 뒤로 순차적으로 스캔하는 방식
  - 테이블 블록 간에는 서로 논리적인 연결고리를 갖고 있지 않으므로 대신 익스텐트 맵을 이용한다
  - 익스텐트 맵은 각 익스텐트의 첫번째 블록 주소 값을 갖는다
  - 읽어야 할 익스텐트 목록을 익스텐트 맵에서 얻고, 각 익스텐트의 첫 번째 블록 뒤에 연속해서 저장된 블록을 순서대로 읽으면 그것이 곧 Full Table Scan
- 랜덤 액세스 Random accsess
  - 논리적, 물리적인 순서를 따르지 않고 레코드 하나를 읽기 위해 한 블록씩 접근하는 방식

#### 1.3.5 논리적 I/O vs 물리적 I/O
- DB 버퍼 캐시
  - 디스크 I/O가 SQL 성능을 결정
  - SQL을 수행하는 과정에 계속해서 데이터 블록을 읽는데, 자주 읽는 블록을 매번 디스크에서 읽는 것은 매우 비효율적
  - 모든 DBMS에 데이터 캐싱 메커니즘이 필수인 이유
  - 공유 메모리 SGA 구성요소
    - 라이브러리 캐시: SQL과 실행계획, DB 저장형 함수/프로시저 등을 캐싱하는 코드캐시
    - DB 버퍼캐시: 데이터를 캐싱, 디스크에서 어렵게 읽은 데이터 블록을 캐싱해 둠으로써 같은 블록에대한 반복적인 I/O call을 줄이는 데 목적이 있음
      - 서버 프로세스와 데이터 파일 사이에 DB 버퍼 캐시가 있음
      - 데이터 블록을 읽을 땐 항상 DB 버퍼 캐시부터 탐색
      - 캐시에서 블록을 찾는다면 프로세스가 I/O call을 하지 않아도 됨
      - 같은 블록을 두 번 읽을때부터는 I/O call을 하지 않아도 됨
      - 버퍼캐시는 공유메모리 영역이므로 같은 블록을 읽는 다른 프로세스도 득을 봄

- 논리적 블록 I/O
  - SQL을 처리하는 과정에서 발생한 총 블록 I/O를 의미
  - SQL을 수행하면서 읽은 총 블록 I/O
  - 논리적 I/O를 줄이는 방법
    - SQL을 튜닝해서 읽는 총 블록 개수를 줄이면 됨
    - 논리적 I/O는 항상 일정하게 발생하지만, SQL 튜닝을 통해 줄일 수 있는 통제 가능한 내생변수
    - SQl을 튜닝해서 논리적 I/O를 줄이면 물리적 I/O도 줄고, 그만큼 성능도 향상됨
- 
- 물리적 블록 I/O
  - 디스크에서 발생한 총 블록 I/O
  - SQL 처리 도중 읽어야 할 블록을 버퍼 캐시에서 찾지 못할 때만 디스크를 액세스 하므로 논리적 블록 I/O 중 일부를 물리적으로 I/O함
  - 요약하면, bchr공식을 이루는 물리적 I/O는 통제 불가능한 외생변수
  - 메모리를 증설해서 DB 버퍼캐시 크기를 늘리는 방법 외에 이것을 직접 줄이는 방법은 없음

#### 1.3.6 Single Block I/O vs Multiple I/O
- Single Block I/O
  - 한 번에 한 블록씩 요청해서 메모리에 적재하는 방식
  - 인덱스를 이용할 때 기본적으로 인덱스와 테이블 모두 Single Block I/O 방식 사용
  - 인덱스는 소량 데이터를 읽을 때 주로 사용하므로 이 방식이 효율적
- Multiblock I/O
  - 많은 벽돌을 실어 나를 때 손수레를 이용하는 것처럼 한 번에 여러 블록씩 요청해서 메모리에 적재하는 방식
  - 많은 데이터 블록을 읽을 때 효율적
  - 인덱스를 이용하지 않고 테이블 전체를 스캔할 때 사용
  - 테이블이 클수록 Single Block I/O 단위도 크면 좋음 -> 프로세스가 잠자는 횟수를 줄여줌
    - 읽고자하는 블록을 db 버퍼캐시에서 찾지 못하면 해당 블록을 디스크에서 읽기 위해 I/O call을 함
    - 그동안 프로세스는 대기 큐에서 잠을 잠
    - 대용량 테이블이면 수 많은 블록을 디스크에서 읽는 동안 여러 차례 잠을 잘텐데, 기왕에 잠을 자려면 한꺼번에 많은 양을 요청해야 잠자는 횟수를 줄이고 성능을 높일 수 있음
    - 대용량 테이블을 Full Scan할 때 Multiblock I/O 단위를 크게 설정하면 성능이 좋아지는 이유
  - 캐시에서 찾지 못한 특정 블록을 읽으려고 I/O call할 때 시스크 상에 그 블록과 인접한 블록들을 한꺼번에 읽어 캐시에 미리 적재하는 기능
    - 인접한 블록: 같은 익스텐트에 속한 블록을 의미
      - Single Block I/O 방식으로 읽더라도 익스텐트 경계를 넘지 못한다는 뜻
  - DBMS 블록 사이즈가 얼마건 간에 OS 단에서는 보통 1MB 단위로 I/O를 수행(OS마다 다름0)

#### 1.3.7 Table Full Sacn vs. Index Range Scan
- 테이블에 저장된 데이터를 읽는 방식
  - 테이블 전체를 스캔해서 읽는 방식 = Table Full Scan
    - 테이블에 속한 블록 전체를 읽어서 사용자가 원하는 데이터를 찾는 방식
    - 시퀀셜 액세스와 Multiblock I/O 방식으로 디스크 블록을 읽음
      - 수십~수백건의 소량 데이터를 찾을 때 수백만~수천만건의 데이터를 스캔하는 것 -> 비효율적 =>큰 테이블에서 소량 데이털르 검색할때는 반드시 인덱스 이용
    - 한 블록에 속한 모든 레코드를 한 번에 읽어 들이고, 캐시에서 못찾으면 I/O Call을 통해 인접한 수십~수백개의 블록을 한꺼번에 I/O하는 메커니즘
    - 스토리지 스캔 성능이 좋아지는 만큼 성능도 좋아짐
  - 인덱스를 이용해서 읽는 방식 = Index Range Scan = 인덱스를 이용한 테이블 액세스
    - 인덱스에서 일정량을 스캔하면서 얻은 ROWID로 테이블 레코드를 찾아가는 방식
      - ROWID: 테이블 레코드가 시크으 상에 어디 저장됐는지를 가리키는 위치 정보
    - 앤덤 액세스와 Single Block I/O 방식으로 디스크 블록을 읽음
    - 캐시에서 블록을 못찾으면 레코드 하나를읽기 위해 매번 I/O Call하는 메커니즘
    - 따라서 많은 데이터를 읽을 때는 Table Full Scan보다 불리함
    - 스토리지 스캔 성능이 수십배 좋아져도 성능이 조금밖에 좋아지지 않음
    - 읽었던 블록을 반복해서 읽는 비효율이 있음
    - 많은 데이터를 일긍ㄹ때 물리적인 블록 I/O뿐만 아니라 논리적인 블록 I/O 측면에서도 불리함 -> 각 블록을 단 한번 읽는 Table Full Scan보다 훨씬 불리함
    - 인덱스는 큰 테이블에서 아주 적은 일부 데이터를 빨리 찾기 위한 도구일 뿐
    - 모든 성능 문제를 인덱스로 해결하려 해선 안됨
    - 읽을 때 데이터가 일정량을 넘으면 인덱스보다 Table Full Scan이 유리
  
#### 1.3.8 캐시 탐색 메커니즘
- Direct Path I/O 제외한 모든 블록 I/O는 메모리 버퍼 캐시를 경유함
- 메모리 공유자원에 대한 액세스 직렬화
  - 버퍼캐시: SGA 구성요소이므로 버퍼캐시에 캐싱된 버퍼블록은 모두 공유자원
  - 공유자원: 모두에게 권한이 있기 때문에 누구나 접근 가능
  - ㄴ 문제는 하나의 버퍼블록을 두 개 이상 프로세스가 동시에 접그하려고 할때마다 발생 -> 동시에 접근하면 블록 정합성에 문제가 생길 수 있음
  - 자원을 공유하는 것처럼 보여도 내부에선 한 프로세스씩 순차적으로 접근하도록 구형해야 함 -> 직렬화 메커니즘 필요
  - 래치 Latch: 줄서기가 가능하도록 지원하는 메커니즘
    - SGA를 구성하는 서브 캐시마다 별도의 래치가 존재
    - 버퍼캐시에는 캐시버퍼 체인 래치, 캐시버퍼 LRU체인 래치 등이 작동
    - 빠른 데이터베이스를 구현하려면 버퍼캐시 히트율을 높여야 하지만, 캐시 I/O도 생각만큼 빠르지 않을 수 있음 -> 이들 래치에 의한 경합이 생길 수 있기 때문
  - 버퍼블록 자체에도 직렬화 메커니즘 존재 -> 버퍼 Lock
  - 이런 직렬화 메커니즘에 의한 캐시 경합을 줄이려면, SQL 튜닝을 통해 쿼리 일량(논리적I/O) 자체를 줄여야 함 
  - 


















