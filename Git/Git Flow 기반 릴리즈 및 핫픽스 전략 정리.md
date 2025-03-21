---
title: "Git Flow 기반 릴리즈 및 핫픽스 전략 정리"
date: 2025-03-21
category: git
tags: [git, git-flow, release, hotfix, 브랜치전략]
---

일자 : 2025.03.21.금

**📌 TIL: Git Flow 기반 릴리즈 및 핫픽스 전략 정리**

## 🛠 문제 상황

- 첫 배포를 준비하는 과정에서 릴리즈 브랜치 및 버전 관리를 어떻게 해야 할지 고민이 생겼습니다.

- 현재 프로젝트는 팀원이 혼자지만, 향후 협업 및 유지보수를 고려해 Git 브랜치 전략을 명확히 세워두는 것이 중요하다고 판단했습니다.

- 이미 main, dev, feature/\* 브랜치로 작업 중이었기에 자연스럽게 Git Flow 전략을 선택해 배포를 준비하게 되었습니다.

## 🌱 Git Flow란?

> Git Flow는 기능 개발, 릴리즈 준비, 긴급 수정 등을 각 브랜치로 분리해 체계적으로 소프트웨어 버전을 관리하는 전략입니다.

### 주요 브랜치 종류

|   브랜치   | 역할                                                           |
| :--------: | -------------------------------------------------------------- |
|    main    | 실제 배포된 안정적인 코드                                      |
|    dev     | 기능 통합 및 테스트가 진행되는 브랜치                          |
| feature/\* | 개별 기능 개발용 브랜치 (dev에서 분기)                         |
| release/\* | 릴리즈 준비 및 QA 진행용 브랜치 (dev에서 분기)                 |
| hotfix/\*  | 배포 후 발생한 버그를 긴급하게 수정하는 브랜치 (main에서 분기) |

### 🌳 Git 브랜치 트리 구조 (Git Flow 스타일)

```
main
  ▲
  └── release/1.0.0  ← QA 및 릴리즈 준비
         ▲
         └── dev  ← 기능 통합
             ▲
             ├── feature/login
             └── feature/profile-edit
```

#### 긴급 수정 시

```
main
  └── hotfix/login-crash
```

## 🛠 릴리즈 브랜치 생성 및 배포 프로세스

1. dev에서 release 브랜치 생성

   > git checkout dev

   > git pull origin dev

   > git checkout -b release/1.0.0

   > git push origin release/1.0.0

2. 릴리즈 QA 및 수정

- 간단한 수정은 release 브랜치에서 직접 수정해도 무방합니다.

- 팀원이 많고 QA 중 여러 명이 동시에 수정하는 경우에는 release 브랜치 기준으로 작업 브랜치 분기합니다.

  > git checkout release/1.0.0

  > git checkout -b fix/release-1.0.0-login-align

이후 PR로 release 브랜치에 병합합니다.

3. main에 릴리즈 반영 및 태깅

   > git checkout main

   > git pull origin main

   > git merge release/1.0.0

   > git tag 1.0.0

   > git push origin main --tags

4. dev에도 반영

   > git checkout dev

   > git merge release/1.0.0

   > git push origin dev`

## 핫픽스와 릴리즈 전 수정의 차이점

| 구분        | 릴리즈 전 수정              | 핫픽스 수정          |
| ----------- | --------------------------- | -------------------- |
| 기준 브랜치 | release/\*                  | main                 |
| 시점        | 아직 배포 전 (QA 중)        | 배포 후 문제 발생 시 |
| 목적        | 기능 안정화, QA 피드백 반영 | 긴급한 버그 수정     |
| 병합 대상   | main + dev                  | main + dev           |
| 브랜치 명   | fix/release-버전-내용 등    | hotfix/버전-내용     |

### 핫픽스 브랜치 작업 예시

1. 핫픽스 브렌치 분기

   > git checkout main

   > git pull origin main

   > git checkout -b hotfix/login-crash

2. 수정 후 커밋

   > git commit -m "🚑 fix: 로그인 크래시 문제 수정"

3. main에 병합

   > git checkout main

   > git merge hotfix/login-crash

   > git tag 1.0.1

   > git push origin main --tags

4. dev에도 병합

   > git checkout dev

   > git merge hotfix/login-crash

   > git push origin dev

## 마무리

혼자 개발하더라도 브랜치 전략을 명확히 정해두면, 히스토리 관리와 유지보수에 큰 도움이 된다는 것을 느꼈습니다.

향후 팀원이 늘어날 때를 대비해 Git Flow 전략을 기반으로 지속적인 브랜치 관리 방식을 이어갈 계획입니다.
