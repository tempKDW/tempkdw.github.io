---
title: "Mixin class 사용시 app 간 계층 구조"
description: ""
date: 2023-12-18T14:43:45+09:00
tags: ["python", "django", "clean code", "refactoring"]
author: "Dongwook Kim"

draft: false
---

앞서 포스팅한 [circular import 에 대한 생각](https://tempkdw.github.io/posts/circular_import/) 에서 해당 문제가 발생한 경우, 계층 구조를 먼저 생각해보자는 이야기를 했었다.

이번에는 이어서 class 의 공통된 기능을 뽑아낼 때 계층 구조를 잘 잡기 위한 방법을 생각해보았다.

일반적으로 Mixin class 는 하위 클래스들의 공통을 묶어 상위 abstract 로 뽑는데 사용되나, 가끔은 상위 클래스들에 공통된 기능을 붙이는 용도로도 사용된다.

django 로 가정하고 예시를 든다.

```python
# A app
class A(models.Model):
    nickname = models.CharField(...)

    def validate_nickname(self):
        ...

# B app
class B(models.Model):
    nickname = models.CharField(...)

    def validate_nickname(self):
        ...
```

이렇게 공통된 field 와 method가 있는 경우, 파편화를 막기 위해 abstract model 을 이용하여 상위의 abstract class 를 만든다.

```python
# C app
class NicknameInfo(models.Model):
    class Meta:
        abstract = True

    nickname = models.CharField(...)

    def validate_nickname(self):
        ...


# A app
from C import NicknameInfo

class A(NicknameInfo):
    ...

# B app
from C import NicknameInfo

class B(NicknameInfo):
    ...
```

이제 nickname 관련 feature 는 `NicknameInfo` 한군데만 고치면 A, B class 모두 영향을 받게 되었다.

계층 구조를 그려보면 다음과 같다.

```
    C(NicknameInfo)
   /  \
 A(A)  B(B)
```

이 경우 A, B app 은 C 에 의존하고 있다. 각 A, B 앱에 import 된 C를 보면 알 수 있다. 이는 A, B 앱을 쓰는 모든 곳에서 Nickname 을 사용할 수 있다고 상정한 것이다.

하지만 만약에 어떤 앱에서 A, B 앱의 여러 model을 호출하는 기능을 추가하고 싶다면 어떻게 해야할까?

예를 들어 A, B 를 이용하여 score 를 매기고 그 score 를 특정 앱에서만 쓰는 경우를 생각해보자.

```
    C(NicknameInfo)
   /  \
 A(A)  B(B)
   \  /
    D(Scorable)
```

A, B 를 참조하므로 D는 당연히 A, B 에 의존성을 가지게 된다. 하지만 `Scorable` 이라는 공통 class 로 묶어내고 싶다면 A, B 에서 D 앱 내의 model 을 상속 받을 수 밖에 없다.

```python
# D app
class Scorable(models.Model):
    class Meta:
        abstract = True

    necessary = models.Field
    field = models.Field
    items = models.Field

    def score(self):
        return len(necessary) + len(field) + len(items)

# A app
from D import Scorable

class A(Scorable):
    necessary = models.Field
    field = models.Field
    items = models.Field
    ...

# B app
from D import Scorable

class B(Scorable):
    necessary = models.Field
    field = models.Field
    items = models.Field
    ...
```

이렇게 된다면 개념도 상으로는 D 가 A, B 를 의존하기를 기대했지만 실상은 A, B 가 D를 의존하게 된다. 추후 D 에서 A 나 B 의 다른 model 을 참조하려 한다면 쉽게 순환 참조 오류를 만날 수 있다.

그렇기에 이런 경우는 A, B 에서 D 를 상속받을게 아니라 오히려 D에서 A, B 를 상속받아야 순서상 옳다.

```python
# D app
from A import A
from B import B

class Scorable:
    neccssary: list[str]
    field: list[int]
    items: list[boolean]

    def score(self):
        return len(necessary) + len(field) + len(items)


class ScorableA(Scorable, A):
    ...

class ScorableB(Scorable, B):
    ...
    
# A app
class A(models.Model):
    ...

# B app
class B(models.Model):
    ...
```

이로써 A, B 앱 내에서는 D 와 아무런 의존성이 없고, D 에서는 `ScorableA` 혹은 `ScorableB` 만 사용하면 `Scorable`의 기능을 사용할 수 있도록 의존성 정리가 되었다.

또한 D 에서 추가로 A, B 앱 내의 다른 model 이 필요한 경우에도 순환 참조 위험 없이 import 할 수 있게 되었다.