---
layout: post
title: "깃허브 블로그 수정기 1부: GitHub Pages 배포 오류와 baseurl 문제 해결"
categories:
- Tech
tags:
- Github
- Jekyll
- Ruby
- GitHub Pages
lang: ko
---

깃허브 블로그를 유지보수하다 보면, 글 내용보다 먼저 **배포와 경로 문제**가 발목을 잡을 때가 있습니다. 이번 1부에서는 실제로 겪었던 GitHub Pages 배포 오류와 잘못 잡힌 baseurl 문제를 정리해보겠습니다.

<br /><br />

![GitHub Pages 배포와 baseurl 수정 요약](/assets/img/posts/2026-05-27-github-blog-fix/2026-05-27-part1-deploy-path-fix-v1.png)

<br /><br />

# 문제의 시작
처음에는 단순히 배포가 실패하는 줄 알았습니다. 그런데 확인해보니 문제는 더 복합적이었습니다.

- GitHub Pages 배포가 정상적으로 끝나지 않음
- CSS / JS 경로가 이상하게 생성됨
- `user pages`인데 `project pages`처럼 URL이 붙음

예를 들어 아래처럼 잘못된 경로가 만들어졌습니다.

```text
https://fkrhtmq123.github.io/pages/fkrhtmq123/assets/css/style.css
```

원래는 아래처럼 나와야 했습니다.

```text
https://fkrhtmq123.github.io/assets/css/style.css
```

즉, 사이트가 **사용자 페이지인데 프로젝트 페이지처럼 해석되는 상태**였습니다.

<br /><br />

# `_config.yml` 수정
가장 먼저 손본 곳은 `_config.yml`입니다.

```yml
url: "https://fkrhtmq123.github.io"
baseurl: ""
repository: fkrhtmq123/fkrhtmq123.github.io
assets: /assets
```

핵심은 두 가지였습니다.

1. `url`을 실제 GitHub Pages 주소로 명시
2. `baseurl`을 빈 문자열로 두어 불필요한 경로가 붙지 않게 처리

이 수정으로 블로그 내부의 정적 리소스 경로와 링크 생성 방식이 사용자 페이지 기준으로 맞춰졌습니다.

<br /><br />

# Workflow도 함께 정리
배포 workflow 쪽에서도 GitHub Pages 빌드 환경을 분명하게 지정했습니다.

```yaml
env:
  JEKYLL_ENV: production
  GITHUB_PAGES: true
  PAGES_REPO_NWO: fkrhtmq123/fkrhtmq123.github.io
```

이 값들은 Jekyll이 GitHub Pages 환경을 올바르게 인식하도록 돕습니다. 특히 `PAGES_REPO_NWO`는 repo 기준 경로 판단에 중요했습니다.

또한 배포가 실제로 성공했는지 단순히 감으로 판단하지 않고, 다음 항목들을 직접 확인했습니다.

- workflow run 상태
- artifact 업로드 성공 여부
- Pages 설정값
- rerun 후 결과

<br /><br />

# 확인 결과
수정 후에는 다음이 정상으로 바뀌었습니다.

- CSS / JS 경로가 올바르게 생성됨
- 글 링크가 user pages 기준으로 출력됨
- GitHub Actions에서 artifact 업로드 성공
- deploy workflow가 정상 실행됨

즉, 이번 문제는 코드 자체보다 **Pages 설정과 Jekyll 경로 인식 문제**에 가까웠습니다.

<br /><br />

# 1부를 마치며
이번 1부는 블로그 글쓰기보다 먼저 해결해야 했던 기반 작업들입니다.

- GitHub Pages 배포 문제 확인
- user pages / project pages 경로 차이 파악
- `_config.yml`의 `url`, `baseurl` 수정
- workflow 환경값 정리

2부에서는 이 배포 문제를 해결한 뒤에 남아 있던 **다국어 페이지네이션과 언어 전환기 문제**를 정리해보겠습니다.
