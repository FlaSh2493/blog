---
emoji: 🧢
title: Git Hooks & Git Husky
date: '2024-05-24 00:00:00'
author: 남승철
tags: git-hooks
categories: 문제해결
---


# Git Hooks

Git에서는 특정 작업이 발생할 때 사용자 지정 스크립트를 실행하여 정해진 이벤트를 발생시키는 방법을 제공합니다.  
여기서 특정 작업이란 Git lifecycle에서의 동작(commit, push 등)을 의미하며, 이를 **Git Hooks**라고 합니다.

---

## Client-Side Hooks & Server-Side Hooks

### Client-Side Hooks

Client-side hooks는 git push 이전의 라이프사이클에서 발생되는 모든 Git 동작 단계에서 사용할 수 있는 hooks입니다.

몇 가지 예시:

- **pre-commit**: commit 바로 직전에 실행
- **prepare-commit-msg**: commit 메시지를 수정하고 변경을 반영하기 직전에 실행
- **commit-msg**: commit 메시지가 들어 있는 임시 파일 경로를 아규먼트로 받으며, 스크립트가 0이 아닌 값을 반환하면 commit되지 않음  
  (프로젝트 상태나 commit 메시지 포맷 검증 시 사용)
- **post-commit**: commit 완료 직후 실행

[상세 hooks 내용](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)

---

### Server-Side Hooks

Server-side hooks는 git push 이후 Git server에서 동작합니다.

몇 가지 예시:

- **pre-receive**: push 요청 시 즉시 수행, non-zero 반환 시 push 거부
- **update**: 각 branch마다 triggering 되며 특정 branch만 reject 처리 가능
- **post-receive**: push 처리 완료 직후 수행, 메일 발송, CI 서버 트리거링 등 활용

[상세 hooks 내용](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)

---

## Git Hooks 사용법

Git hooks는 리포지토리 생성 시 자동 생성되며, `.git/hooks/` 경로에 sample 파일들이 존재합니다.

```text
.git/hooks/
  applypatch-msg.sample     
  fsmonitor-watchman.sample 
  pre-applypatch.sample     
  pre-merge-commit.sample   
  pre-rebase.sample         
  prepare-commit-msg.sample 
  update.sample
  commit-msg.sample         
  post-update.sample        
  pre-commit.sample         
  pre-push.sample           
  pre-receive.sample        
  push-to-checkout.sample
```

- 실제 동작하는 hooks 파일은 `.sample` 제거 또는 새 파일 생성
- Git repository clone 시 hooks는 복제되지 않음
- [참고](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)

---

## Git Husky

Git hooks를 쉽게 사용할 수 있는 라이브러리

### 설치

```bash
yarn add --dev husky
```

Monorepo 기준 설치:

```jsonc
// ts/package.json
"prepare": "cd .. && husky install ts/.husky"
```

- 설치 시 `.husky/` 폴더 생성

```text
.husky/
  _/
    .gitignore
    husky.sh
```

- `husky.sh`: 작성한 hooks 파일을 Git hooks에 연결

---

## Hooks 생성 예시

### pre-commit

```sh
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

changed_list=$(git diff --cached --name-only)
cd ts

if [ ! -f diffCache ]; then
  touch diffCache
fi

echo $changed_list >> diffCache
```

- 변경사항 있는 파일 목록을 `diffCache`에 저장 -> (이제보니 HEAD와 비교해도 되는데..)
- pre-push 단계에서 사용

---

### pre-push

```sh
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

cd "ts/" || exit

if [ ! -f diffCache ]; then
	touch diffCache
fi

changed_list=$(<diffCache)
exit_code=$?

if [[ $changed_list == *"/app1/"* ]]; then
    yarn app1:test
fi
if [[ $changed_list == *"/app2/"* ]]; then
  yarn app2:test
fi
if [[ $changed_list == *"/packages/"* ]]; then
    yarn packages:test
fi

if [[ $exit_code = 0 ]]; then
  > diffCache
fi
```

- pre-commit에서 저장한 `diffCache`를 불러와 변경된 product 테스트만 실행
- 테스트 성공 시 `exit_code === 0` 반환, 실패 시 push 거부
- 성공 후 `diffCache` 초기화

## 효과
- 모노레포에서 여러 팀이 관리하는 app들 중에서 변경사항이 있는 app의 테스트만 선별적으로 실행할 수 있는 구조가 완성됨.

---

## 참고자료

- [Husky 공식문서](https://typicode.github.io/husky/)
- [RedHat Git Hooks](https://www.redhat.com/sysadmin/git-hooks) (검색 키워드: git hooks, pre-push, run test has changed)
- [Git Hooks Docs](https://git-scm.com/docs/githooks)
- [Pro Git Book: Customizing Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
