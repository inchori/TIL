### Tricolor Mark and Sweep Algorithm
---
- 흰색 집합(White Set): 프로그램에서 더 이상 접근할 수 없어서 GC 대상이 되는 객체
- 검은색 집합(Black Set): 프로그램이 사용하고 있고, white set의 객체에 대한 참조가 없는 객체
- 회색 집합(Grey Set): 프로그램이 사용하고 있고 흰색 집합의 객체를 가리킬 수도 있어서 검사를 진행해야하는 객체

**Tricolor Mark and Sweep Algorithm 순서**
1. GC가 모든 객체를 흰색로 칠한다.
2. GC가 모든 루트 객체를 방문해서 회색으로 칠한다.
3. GC는 회색 집합의 객체를 하나씩 검은색으로 변경하면서 그 객체가 가리키는 흰색 집합 객체 확인되면 회색으로 칠한다
4. 하위 객체까지 완료되면 검은색으로 칠한다.
5. 흰색 집합은 더 이상 접근할 수 없다고 판단하고 메모리를 해제한다.

**Concurrent Mark and Sweep**

- Go언어는 STW(Stop the World)로 인한 GC Latency를 낮추기 위해 CMS(Concurrent Mark and Sweep) 방식 채택
- Mark and Sweep 과정에서 STW 상태에 들어가지 않고 애플리케이션과 동시에 작업
- 동시성 환경에서 발생하는 문제를 방지하기 위해 write barrier를 사용하여 marking 단계에서 생성되는 객체는 바로 grey 표시
- 메모리 단편화의 영향으로 메모리 할당 속도가 느려지는 단점
- Go의 GC는 가용 가능한 CPU 사용률의 30%를 목표로 CPU 리소스를 사용 (GC worker 25%, assist goroutine 5%)

### Non-Generational GC
---
- 세대별 가설: 새로 생긴 객체는 오래된 객체에 비해 일찍 죽는다는 가설
- Young Generation: 힙을 새로 생성된 객체 저장
- Old Generation: 오래된 객체를 관리
- Minor GC: Young Generation에서 더 자주 GC를 수행
- Major GC: Young Generation에서 여러 번 살아남은 객체를 Old Generation으로 승격 후 기존 OG에서 GC 수행

**Go는 왜 Non-Generational GC를 수행할까?**

- Write barrier에 대한 오버헤드를 허용할 수 없기 때문
- Write barrier는 OG의 객체가 YG의 객체를 참조할 때 세대 간 참조 사실을 Card table에 기록하는 역할을 하는 메커니즘
- Go언어는 컴파일 시점에 escape analysis를 통해 최적화 수행

**Go Compiler Escape Analysis**

- Go 컴파일러는 escape analysis 단계에서 객체가 함수 범위를 벗어나는지, 참조 전달하는지 지역성을 확인하여 데이터를 스택 or 힙에 할당할 수 있는지 결정
- Go는 최대한 스택 영역을 활용하기 위해 value-oriented 방식을 사용
- Go는 수명이 짧다고 판단되는 객체는 힙이 아닌 스택에 할당하여 GC를 수행할 필요도 없이 스택 영역에 할당되었다가 사라짐

### Non-Compaction
---
- GC는 압축 방식과 비압축 방식을 사용
- 압축 방식
	- Alive 상태의 객체를 힙의 끝으로 재배치하여 힙을 압축
	- 메모리 파편화 방지
	- 빠른 메모리 할당
- 비압축 방식
	- GC 후 힙 메모리의 객체를 재배치하지 않음
	- 메모리 할당과 GC를 반복하면 힙 메모리가 파편화되어 성능 악화

**TCMalloc (Thread Caching Memory Allocation)**

- 구글에서 개발한 메모리 할당 아리브러리
- 중앙 힙(central heap)과 각 스레드 별로 스레드 로컬 캐시(thread local cache)라는 영역을 제공하여 관리
- TLC가 있기 때문에 메모리 할당 시 불필요한 동기화가 줄어들어 lock에 대한 비용 감소

### GC 옵션
---
- `GOGC`는 기존 메모리 대비 얼만큼의 메모리가 할당되면 GC가 실행될지에 대한 백분율 값 설정하는 환경 변수 (기본값 = 100)

```go
GOGC=200 go build .
```

- 기존 메모리 대비 200%의 메모리가 할당되면 GC 수행