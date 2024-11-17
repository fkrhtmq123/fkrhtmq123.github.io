---
layout: post
title: 깃허브 블로그 생성
categories: 
- Tech
tags:
- Github
- Ruby
lang: ko
---

필자는 개발자로 일하면서 전 직장까지하면 3년이 되는데 개발자로서 기술 블로그가 없으면 안될꺼 같아 여러 블로그 사이트를 확인하다가 깃허브로 블로그를 생성 할수도 있다고 해서 깃블로그를 작성하여 어떻게 생성하는지 글을 쓸려고 한다.

<br /><br />

# 깃 프로젝트 만들기
깃으로 블로그를 만들기 위해서는 먼저 깃 프로젝트를 생성해야합니다.

<img src="/assets/img/git_blog_create_project.png">

[Repository name] 에는
```sh
[USERNAME].github.io
```

<br /><br />

# 깃 클론
위에서 생성한 깃 프로젝트를 클론을 합니다.

<img src="/assets/img/git_blog_clone.png">

생성한 프로젝트 페이지에서 초록색 버튼으로 ```Code```를 클릭 하여 ```HTTPS```에서 깃 주소를 복사
<br />
복사한 주소를 클론을 하고 싶은 폴더로 이동하여 클론

```sh
cd developer
git clone https://github.com/fkrhtmq123/fkrhtmq123.github.io.git
```

<br /><br />

# 깃 블로그 테마 정하기
[http://jekyllthemes.org/](http://jekyllthemes.org/)
<br />
위 주소에서 쓰고 싶은 테마를 정해서 클릭

<img src="/assets/img/jekyll_theme1.png">

라이센스를 확인하고 다운로드를 클릭해서 테마를 다운로드<br />
라이센스가 무료(MIT)가 아닌 것도 있기 때문에 잘 확인해야합니다.

<img src="/assets/img/jekyll_theme2.png">

<br /><br />

# 깃 테마 적용
다운로드한 깃 테마 알집을 풀어 내용을 전부 위에서 생성하여 클론한 깃 프로젝트에 복사 붙이기.

<img src="/assets/img/jekyll_theme3.png">

<img src="/assets/img/jekyll_theme4.png">

<br /><br />

# 깃 커밋 & 푸쉬
깃에 커밋과 푸쉬를 하면 테마가 적용, 적용되는데는 빠른 테마가 있고 느린 테마가 있습니다.<br />
적용하고 약 1분 정도 기다립니다.

이걸로 깃 블로그가 완성.

다음 글에는 깃 블로그를 로컬에서 사용하는 방법을 설명하겠습니다.