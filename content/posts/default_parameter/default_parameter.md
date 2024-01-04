---
title: "default value parameter 가 가독성에 미치는 해악"
description: "default parameter 는 refactoring 할 때만 씁시다."
date: 2023-12-18T14:43:45+09:00
tags: ["파이썬", "클린코드", "리팩토링", "python", "default parameter", "clean code", "refactoring"]
author: "Dongwook Kim"

draft: false
---

default value parameter 는 함수에 파라미터를 지정할 때 자주 쓰이는 방법이다.
```python
def foo(a: bool = True):
    if a:
        return "foo"
    else:
        return "bar"
```

문제는 이 방법이 함수의 제작자의 경우에는 명시적으로 느껴지지만 사용자 혹은 독자의 경우에는 굉장히 묵시적이라는데 있다.

```python
# default argument
>> foo()
"foo"

# positional argument
>> foo(True)
"foo"

# keyword argument
>> foo(a=False)
"bar"
```

default value parameter 는 positional / keyword argument 외에도 하나의 방법을 더 늘려주는데, 명시적으로 argument 를 넘기지 않아도
이미 지정된 default value 를 함수 내에서 사용할 수 있게 끔 한다.

명시성은 [The Zen of Python](https://peps.python.org/pep-0020/) 에 2번째로 기재될 만큼 python 에서 중요하다.

위에서 언급한대로 default value parameter 는 함수의 사용자, 혹은 독자로 하여금 함수가 어떤 parameter 를 받는지 알 수 없게 한다.

`foo()` 만 놓고 보자면 이 함수의 제작자나 코드를 읽어본 사람이 아니면 parameter 가 없는 함수로 보이기 때문이다.

함수의 parameter 는 어떤 기계의 컨트롤 패널이다.

![컨트롤 패널](../images/control_panel.png)

parameter 는 적으면 적을 수록 좋고, 내가 조작하고 있는게 뭔지 어떤 값을 갖게 되는지 명확하게 아는게 좋다.

그렇다면 default parameter 는 언제 쓰이면 좋을까?

이미 사용처가 있는 함수에 호환성을 유지하며 parameter 를 추가하고자 할때이다.

이미 사용처가 있는 함수는 손쉽게 건드리기 어렵다. unit test 가 있는 상황이라면 좋겠지만 그렇지 않다면 더더욱이나 쉽지 않다.

그럴 경우, 함수를 호출 하는 쪽을 모두 수정하지 않고 parameter 를 늘리기 가장 쉬운 방법이 default value parameter 를 주는 것이다.

이전에 foo 를 return 하는 함수가 있다고 가정해보자.

```python
def foo():
    return "foo"

# 어딘가
foo()
```

이제 조건에 따라 bar 를 return 하는 함수로 바꾸고 싶을 경우는,

```python
def foo(a=False):
    if a:
        return "bar"
    else:
        return "foo"

# 어딘가
foo()
```

이렇게 호출하는 쪽의 코드를 수정하지 않고도 함수의 parameter 를 추가 할 수 있다.
