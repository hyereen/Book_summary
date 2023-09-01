# PART 1 기본편

## CHAPTER 1 Apache Airflow 살펴보기
### p.4 1.1 데이터 파이프라인 소개
- 방향성 비순환 그래프 Directed Acyclic Graph, DAG
  - 그래프는 화살표 방향성의 끝점을 포함하되 반복이나 순환을 허용하지 않음(비순환)
  - 비순환 속성은 태스크 간의 순환 실행을 방지하기 때문에 매우 중요함