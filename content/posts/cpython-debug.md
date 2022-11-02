---
title: "Python 내부의 C 코드 들여다보기"
date: 2022-10-21T10:52:51+09:00
tags: ["cpython", "lldb", "debug"]
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

> ! 아래의 예제는 모두 macOS Monterey 12.5.1 에서 작성되었다.


먼저 python 내부에 c 코드를 들여다보기 위해서는 cpython 소스를 받아야 한다.

```shell
git clone https://github.com/python/cpython
cd cpython
```

원하는 버전으로 branch 를 바꿔주고 컴파일을 진행한다.

> configure 옵션 [참고](https://docs.python.org/3/using/configure.html)

```shell
git switch 3.9
./configure --with-pydebug --with-openssl=$(brew --prefix openssl)
make -s -j2
```

`make` 시 macOS 에서 `'lzma.h' file not found` 가 발생할 경우 brew 로 xz 를 설치하고 환경변수를 잡아준다.

> 환경마다 path가 다를수 있음에 주의
```
brew install xz
export LDFLAGES="-Lls /opt/homebrew/Cellar/xz/5.2.7/lib $LDFLAGS"
export LDFLAGES="-L/opt/homebrew/Cellar/xz/5.2.7/lib $LDFLAGS"
export CPPFLAGS="-I/opt/homebrew/Cellar/xz/5.2.7/include $CPPFLAGS"
```

정상적으로 완료되었다면 이제 폴더에 생성된 `python.exe` 를 확인할 수 있다.

macOS 에서도 정상적으로 실행이 가능하므로 이를 이용해 `lldb`를 실행한다.

> ! gdb 는 m1 을 아직 지원하지 않는다. (2022.11.02)

```shell
lldb python.exe
```

> [lldb 사용법](https://lldb.llvm.org/use/tutorial.html)

`run` 명령어를 통해 `lldb` 내부에서 프로그램을 실행하여 breakpoint 를 찍을 곳을 가늠해본다.

python의 `dis` lib 을 사용하면 python의 bytecode operation 을 확인 할 수 있어, 어느 부분부터 접근해야 하는지 알 수 있다.

```shell
(lldb) run
>>> from enum import Enum
>>> from dis import dis
>>>
>>> class Foo(str, Enum):
>>>     X = "X"
>>> 
>>> def cmp():
>>>     return Foo.X == "X" 
>>> 
>>> dis(cmp)
  2           0 LOAD_GLOBAL              0 (Foo)
              2 LOAD_ATTR                1 (X)
              4 LOAD_CONST               1 ('X')
              6 COMPARE_OP               2 (==)
              8 RETURN_VALUE
```
위의 `dis` 가 보여준 내용으로 strEnum 과 str 의 비교는 `COMPARE_OP` 에서 수행하는 것을 알 수 있다.
> ! `SIGINT`(Ctrl+c) 로 프로그램에서 `lldb` 로 돌아갈 수 있다.

cpython 소스에서 검색하여 위치를 찾아보자.

```shell
# Python/ceval.c
3004         case TARGET(COMPARE_OP): {
3005             assert(oparg <= Py_GE);
3006             PyObject *right = POP();
3007             PyObject *left = TOP();
3008             PyObject *res = PyObject_RichCompare(left, right, oparg);
3009             SET_TOP(res);
3010             Py_DECREF(left);
3011             Py_DECREF(right);
3012             if (res == NULL)
3013                 goto error;
3014             PREDICT(POP_JUMP_IF_FALSE);
3015             PREDICT(POP_JUMP_IF_TRUE);
3016             DISPATCH();
3017         }
```

위의 부분에서 처리하고 있음을 알 수 있다.

이제 breakpoint 를 설정하여 실제로 프로그램이 구동 될 때 어떤 값들이 들어가 있는지 확인해보자.
`b {source_file_path}:{line_number}`

```shell
(lldb) b Python/ceval.c:3004
Breakpoint 1: where = python.exe`_PyEval_EvalFrameDefault + 41684 at ceval.c:3005:13, address = 0x00000001002506c8
```

설정되었음을 확인하고 다시 프로그램으로 돌아간다.

```shell
(lldb) c
Process 94651 resuming
```

breakpoint 가 걸릴 function을 실행한다.
```
>>> cmp()
Process 94651 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001002506c8 python.exe`_PyEval_EvalFrameDefault(tstate=0x00000001011043b0, f=0x0000000101376c20, throwflag=0) at ceval.c:3005:13
   3002         }
   3003 
   3004         case TARGET(COMPARE_OP): {
-> 3005             assert(oparg <= Py_GE);
   3006             PyObject *right = POP();
   3007             PyObject *left = TOP();
   3008             PyObject *res = PyObject_RichCompare(left, right, oparg);
Target 0: (python.exe) stopped.
(lldb) 
```

정상적으로 breakpoint가 동작하는것을 볼 수 있다.

이제 Pyobject *left 에 들어있는 값을 확인해보자.

n 으로 left 가 assign 된 이후 line 까지 넘어간 뒤,
```shell
Process 14415 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step over
    frame #0: 0x000000010025074c python.exe`_PyEval_EvalFrameDefault(tstate=0x00000001011041d0, f=0x0000000101376c20, throwflag=0) at ceval.c:3008:50
   3005	            assert(oparg <= Py_GE);
   3006	            PyObject *right = POP();
   3007	            PyObject *left = TOP();
-> 3008	            PyObject *res = PyObject_RichCompare(left, right, oparg);
   3009	            SET_TOP(res);
   3010	            Py_DECREF(left);
   3011	            Py_DECREF(right);
Target 0: (python.exe) stopped.
```

print 로 내부 구조를 볼 수 있다.
```shell
(lldb) p *left->ob_type
(PyTypeObject) $42 = {
  ob_base = {
    ob_base = {
      ob_refcnt = 6
      ob_type = 0x0000000101604d50
    }
    ob_size = 0
  }
  tp_name = 0x00000001013a7de0 "Foo"
  tp_basicsize = 96
  tp_itemsize = 0
  tp_dealloc = 0x000000010014e93c (python.exe`subtype_dealloc at typeobject.c:1198)
  tp_vectorcall_offset = 0
  tp_getattr = 0x0000000000000000
  tp_setattr = 0x0000000000000000
  tp_as_async = 0x00000001011293d8
  tp_repr = 0x000000010015fef4 (python.exe`slot_tp_repr at typeobject.c:6651)
  tp_as_number = 0x00000001011293f0
  tp_as_sequence = 0x0000000101129528
  tp_as_mapping = 0x0000000101129510
  tp_hash = 0x00000001001aef18 (python.exe`unicode_hash at unicodeobject.c:11983)
  tp_call = 0x0000000000000000
  tp_str = 0x0000000100160328 (python.exe`slot_tp_str at typeobject.c:6667)
  tp_getattro = 0x0000000100131214 (python.exe`PyObject_GenericGetAttr at object.c:1279)
  tp_setattro = 0x0000000100131e88 (python.exe`PyObject_GenericSetAttr at object.c:1360)
  tp_as_buffer = 0x0000000101129578
  tp_flags = 269243904
  tp_doc = 0x000000010125b670 "An enumeration."
  tp_traverse = 0x000000010015c10c (python.exe`subtype_traverse at typeobject.c:1106)
  tp_clear = 0x000000010015c334 (python.exe`subtype_clear at typeobject.c:1168)
  tp_richcompare = 0x000000010019a464 (python.exe`PyUnicode_RichCompare at unicodeobject.c:11480)
  tp_weaklistoffset = 88
  tp_iter = 0x00000001001af300 (python.exe`unicode_iter at unicodeobject.c:15966)
  tp_iternext = 0x0000000100131860 (python.exe`_PyObject_NextNotImplemented at object.c:1066)
  tp_methods = NULL
  tp_members = 0x00000001011295b0
  tp_getset = 0x0000000100464750
  tp_base = 0x0000000100465178
  tp_dict = 0x00000001013fe830
  tp_descr_get = 0x0000000000000000
  tp_descr_set = 0x0000000000000000
  tp_dictoffset = 80
  tp_init = 0x000000010015352c (python.exe`object_init at typeobject.c:3868)
  tp_alloc = 0x000000010014d478 (python.exe`PyType_GenericAlloc at typeobject.c:1043)
  tp_new = 0x0000000100158dbc (python.exe`slot_tp_new at typeobject.c:6992)
  tp_free = 0x00000001002f2768 (python.exe`PyObject_GC_Del at gcmodule.c:2349)
  tp_is_gc = 0x0000000000000000
  tp_bases = 0x00000001014030f0
  tp_mro = 0x0000000101405470
  tp_cache = NULL
  tp_subclasses = NULL
  tp_weaklist = 0x0000000101405410
  tp_del = 0x0000000000000000
  tp_version_tag = 154
  tp_finalize = 0x0000000000000000
  tp_vectorcall = 0x0000000000000000
}
```

> PyObject 에 대한 내용은 [공식문서 참고](https://docs.python.org/ko/3/c-api/structures.html#c.PyObject)

이제 lldb 의 다양한 명령어들을 사용하여 python 의 내부 동작을 파해쳐볼 수 있다.

Happy Hacking!