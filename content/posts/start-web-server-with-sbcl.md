---
title: "Steel bank common lisp 으로 webserver 띄우기"
date: 2025-05-29T21:41:45+09:00
tags: ["lisp", "bootstrap", "web server"]
author: "Dongwook Kim"
draft: false
---

### emacs 설치
`brew install emacs`

### steel bank common lisp 설치
`brew install sbcl`

### emacs package 관리자 quicklisp 설치
```
$ curl -O https://beta.quicklisp.org/quicklisp.lisp
$ curl -O https://beta.quicklisp.org/quicklisp.lisp.asc
$ sbcl --load quicklisp.lisp
```
중간 중간 나오는 명령어를 따라주면 된다.

### emacs lisp extension 인 slime 설정
quicklisp 에 자체 내장되어 있으므로 이걸 쓰도록 한다.

`$ find ~ -name "swank.lisp"` 로 확인하여 quicklisp 폴더 내에 존재한다면 있는것.

`$ ln -s ~/quicklisp/dists/quicklisp/software/slime-v2.30/ ~/quicklisp/local-projects/slime-v2.30`

`quicklisp/local-projects` 안에 symlink 를 생성하여 주면 된다. 

quicklisp 내장 외에 따로 slime 를 설치할 경우에도 위의 경로에 symlink 를 생성하면 문제가 생기지 않을 것이나 해보지는 않았다.

symlink 생성없이 따로 설치하여 사용하면 충돌 발생하니 주의.

```
# ~/.emacs

(add-to-list 'load-path "~/quicklisp/dists/quicklisp/software/slime-v2.30/")

(require 'slime-autoloads)
;; SLIME 설정
(setq inferior-lisp-program "sbcl")
(setq slime-net-coding-system 'utf-8-unix)

(slime-setup '(slime-fancy))
```
경로는 다를수 있으니 체크

### web server framework 설치
[caveman2](https://github.com/fukamachi/caveman) 를 설치한다

```
$ emacs
M-x slime
* (ql:quickload :caveman2)
* (caveman2:make-project #P"~/quicklisp/local-projects/hello-caveman" :author "tempkdw")
* (ql:quickload :hello-caveman)
* (hello-caveman:start :port 3000)
```

이후 browser 에서 localhost:3000 을 통해 실행되고 있음을 확인할 수 있다.