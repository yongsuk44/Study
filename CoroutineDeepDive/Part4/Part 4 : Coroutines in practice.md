# Kotlin Coroutines in practice

## 목차

- [Common use cases](#part-41--common-use-cases)

---

## [Part 4.1 : Common use cases](Common%20use%20cases.md)

대부분 Application 구조는 다음 3가지 계층으로 나눕니다.

- Data / Adapter Layer
- Domain Layer
- Presentation / UI Layer

### Data / Adapter Layer

데이터 저장, 추출, 변환 등 작업을 합니다.   
즉, 네트워크 요청, 데이터베이스 작업 등을 하는 영역입니다.

여기서 코루틴은 `suspend` 키워드를 통해 각 작업을 비동기적으로 수행할 수 있게 지원해줍니다.  
추가로, `Room` 라이브러리의 경우 `Flow`를 지원하며 쉬운 반응형 UI 구현을 도와줍니다.