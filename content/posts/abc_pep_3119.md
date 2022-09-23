---
title: "PEP 3119 ABC 번역"
description: "한국어 번역"
date: 2022-09-23T14:15:00+09:00
tags: ["python", "pep", "translation"]
author: "Dongwook Kim"

showToc: false
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
searchHidden: true
---
# *PEP 3119 – Introducing Abstract Base Classes*
[original link](https://peps.python.org/pep-3119/)

## 이론적 해석
객체 지향 프로그래밍에서, 객체와 상호작용하는 사용 패턴은 두가지 기본 분류로 나눌 수 있다. 하나는 ‘호출(invocation)’ 이고 다른 하나는 ‘분석(inspection)’ 이다.

호출은 객체의 메소드를 호출하는 것을 의미한다. 대개 다형성과 결합되어 메소드를 호출하면 어떤 타입의 객체이냐에 따라 다른 코드를 실행하게 된다.

```python
class A:
    def foo(self):
         print("A")

class B(A):
    def foo(self):
         print("B")

A().foo()
B().foo()
```

분석은 외부 코드(해당 객체의 메소드 밖의)에서 해당 객체의 타입이나 프로퍼티를 확인하고 확인한 정보에 따라 어떻게 객체를 다룰지 결정하는 
것을 의미한다.

```python
def foo(obj):
    if isinstance(obj, A):
        print("A")
    elif isinstance(obj, B):
        print("B")
```

두가지 사용 패턴은 동일한 결과를 보여주는데, 다양한 처리를 지원할 수 있는 것과 잠재적으로 새로운 객체들을 통일된 방법으로 지원하는 것이다. 또 동시에 각기 다른 타입의 객체를 커스텀할 수 있도록 허용 한다.

고전적인 OOP 이론에서는, 호출이 좀더 자주 쓰이는 사용 패턴이고, 분석은 절차 지향 프로그래밍 스타일의 유산 정도로 생각되어 자주 쓰이지 않는다. 그러나 실제로는 이런 시각은 너무 경직되어있고 독단적이며, 파이썬 같은 동적 특성을 지닌 언어에서는 매우 다른 일종의 설계 경직성으로 이어진다.

특히, 객체 클래스의 제작자가 예상하지 못한 방법으로 객체를 다뤄야하는 일이 자주 발생한다. 객체가 모든 유저의 요구에 만족하도록 모든 객체를 구성하는것은 항상 최고의 해결방법은 아니다. 게다가, 행동이 엄격하게 객체 내에 캡슐화되어야 하는 고전적인 OOP 와는 대조적인 여러가지 강력한 분배 철학들이 있다. 예를 들어 규칙 혹은 패턴 매칭 주도 로직이다.

```ocaml
type document
  = Text
  | Drawing
  | Spreadsheet

fun draw (Text)        = (* draw text doc... *)
  | draw (Drawing)     = (* draw drawing doc... *)
  | draw (Spreadsheet) = (* draw spreadsheet... *)

fun load (Text)        = (* load text doc... *)
  | load (Drawing)     = (* load drawing doc... *)
  | load (Spreadsheet) = (* load spreadsheet... *)

fun save (Text)        = (* save text doc... *)
  | save (Drawing)     = (* save drawing doc... *)
  | save (Spreadsheet) = (* save spreadsheet... *)
```
[oop - How does pattern-match driven logic look like in real world applications? - Stack Overflow](https://stackoverflow.com/questions/30937538/how-does-pattern-match-driven-logic-look-like-in-real-world-applications)

반면, 고전적인 OOP 이론학자들의 분석의 비난점 중 하나는 형식주의의 부족과 분석 대상의 ad hoc 특성이다. 객체의 거의 모든 면을 리플렉트 가능하고 외부 코드로 직접 접근이 가능한 Python 같은 언어에서는, 객체가 특정 프로토콜을 따르는지 아닌지를 확인할 수 있는 여러가지 다른 방법들이 있다. 예를 들어 만약 ‘이 객체는 뮤터플 시퀸스 컨테이너인가?’ 라고 했을때, 해당 객체가 list 를 상속받는지 본다던가, ‘__getitem__’ 이름의 메소드를 가지고 있다던가. 하지만 이런 테스트들이 명확해보임에도 불구하고 전자는 거짓 음성, 후자는 거짓 양성이므로 둘다 틀리다.

```python
def is_mutable_sequence_container(obj):
    return True if isinstance(obj, list) else False

from collections import deque
is_mutable_sequence_container(deque())  # 거짓 음성
```
```python
def is_mutable_sequence_container(obj):
    return True if hasattr(obj, '__getitem__') else False

is_mutable_sequence_container(tuple())  # 거짓 양성
```

일반적으로 합의된 해결법은 테스트들을 표준화하고 공인된 방법으로 묶어서 정렬하는 것이다. 이것은 상속이나 다른 방법을 통해 각 클래스와 테스트 가능한 표준 프로퍼티 집합을 연결함으로써 쉽게 가능하다. 각 테스트는 일련의 약속들을 가지는데, 클래스의 일반적인 동작과 다른 클래스 메소드들로 사용가능하다는 것을 포함한다.
```python
class A:
    def foo(self): pass

class B:
    def foo(self): pass

# TEST: A, B가 동일한 foo 메소드를 가지는가?
# TEST: A, B 가 클래스인가?
```

이 PEP 는 Abstract Base Classes 또는 ABC 로 알려진 테스트들을 구성하는 특정 전략을 제안한다. ABC 는 객체의 특정 기능을 외부 검사자에게 알려주기 위해 객체의 상속 트리에 추가되는 간단한 Python 클래스이다. 테스트는 `isinstance()` 를 써서 가능하며 특정 ABC의 존재는 테스트가 통과했다는 것을 의미한다.

이에 더해, ABC 는 특정 동작을 설정하는 최소한의 메소드 집합을 정의한다. ABC 타입을 기반으로 하는 객체를 구별하는 코드는 이런 메소드들이 항상 존재한다고 신뢰할 수 있다. 이런 각각의 메소드들은 ABC 문서에 설명된 일반화된 추상 시멘틱 정의를 가진다. 이 표준 시멘틱 정의는 강제되는 것은 아니지만 강력히 권장된다.
```python
class A(ABC):
    @abstractmethod
    def foo(self): pass

    @abstractmethod
    def bar(self): pass

# 최소한의 메소드집합 (foo, bar)
```

파이썬의 모든 다른 것들처럼, 이러한 규약은 강제되지 않은 합의인만큼, 언어가 ABC 로 만들어진 일부 규약을 시행하지만 나머지는 클래스의 구현자에게 달려있음을 의미한다.
