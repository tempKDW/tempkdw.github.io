---
title: "Early return에 대한 생각"
description: "분기와 early-return은 다르게 쓰여야 한다."
date: 2021-02-17T20:40:20+09:00
tags: ["python", "coding"]
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

early return 이란 특정 상황에서 function 내 주요 로직이 실행되기 이전에 return을 함으로써 실행 cost를 줄이고 가독성을 높이는 방법이다.

```python
def notify(data):
    if not data.get("to"):
        return False
    if data.get("platform") not in NOTIFY_PLATFORMS:
        return False

    return NOTIFY_PLATFORMS[data["platform"]](data)
```

위의 경우에서 보듯이 알림을 주는 function에서 전처리를 통해 실제 로직이 실행 되기 이전에 return을 주는 것을 알 수 있다.

안좋은 예로 보면

```python
def notify(data):
    if not data.get("to"):
        return False
    elif data.get("platform") not in NOTIFY_PLATFORMS:
        return False
    else:
        return NOTIFY_PLATFORMS[data["platform"]](data)
```

와 같이 if-elif-else 로 모든 로직을 감싸는 경우라고 생각 되는데, 이 경우는 해당 function의 정상 로직이 어느 분기인지 알기 힘들다.

분기로 if-elif-else가 쓰여야 하는 경우를 생각해보면, 

```python
def calc(a, b, op):
    if op == "+":
        return a + b
    elif op == "-":
        return a - b
    elif op == "*":
        return a * b
    elif op == "/":
        return a / b
```

와 같은 경우는 각 분기가 정상 로직으로 op에 의해서 나뉜다. input에 의해 output 이 4가지 경우가 되는 것이다.

하지만 위의 안좋은 예는 오로지 output이 2가지 뿐이다. data가 notify 처리에 적절하지 못하여 처리하지 않고 early-return 한 것과 적절하여 notify 처리 후 return 하는 것 2가지이다.

early-return을 그려보면 아래와 같다.
```bash
|
|\ # early-return
|\ # early-return
|
```

분기는 아래와 같다.
```bash
   |
-------
| | | |
```

그렇기에 if 만을 사용하여 early-return을 구성 하였다면 뒤이어 코드를 읽는 사람은 if를 모두 제외한 로직만 읽으면 function의 메인 로직을 좀더 빠르게 이해할 수 있다.

다시 위의 notify function에서 early-return 으로 쓰인 if 만 걷어낸다면

```python
def notify(data):
    return NOTIFY_PLATFORMS[data["platform"]](data)
```
와 같이 쉽게 메인 로직이 파악 가능하다.

그러므로 function 내에서 로직을 수행하기 이전에 특정 전처리가 필요한 경우 if 만을 사용하여 early-return을 구성하는 것이 가독성에 도움이 된다고 생각한다.