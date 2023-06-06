---
title:   "Github page 생성기"
excerpt: "GitHub github.io 생성기록"
toc: true
toc_sticky: true

categories:
  - Blog
tags:
  - Blog
  - Git Page
  - TOC
last_modified_at: 2023-06-06T11:06:00+09:00
---

# Background
github를 이용하여 개인 홈페이지를 만드는 과정 입니다.

개인 홈페이지를 처음 만드는 분들에게 도움이 되고자 정리합니다!

# Tutorial
## 1. install ruby by rbenv

```bash
$ brew update
$ brew install rbenv ruby-build

$ rbenv install -l
$ rbenv install 3.1.3 # other version occur errors!
$ rbenv global 3.1.3
$ rbenv versions

$ vim ~/.zshrc
$ [[ -d ~/.rbenv  ]] && \
  export PATH=${HOME}/.rbenv/bin:${PATH} && \
  eval "$(rbenv init -)"
$ source ~/.zshrc
```


## 2. install jekyll and bundler
`Bundler` 란 웹 어플리케이션을 구성하는 자원(html, css, javascript etc)등을 각각의 모듈 단위로 나누어 번들형태로 만들어 내는 도구 입니다.    
`Jekyll` 이란 정적 사이트 생성기입니다.

```bash
$ gem install jekyll bundler
$ jekyll -v
$ bundle exec jekyll -v # jekyll 3.2.1

$ bundle add webrick
$ bundle install

# local hosting
$ bundle exec jekyll serve
```

## 3. create repo & setting theme

```bash
$ git clone https://github.com/mmistakes/minimal-mistakes.git
$ mv jekyll/minimal-mistakes/ {GitHub_Page Dir}
$ cd {GitHub_Page Dir}

$ git remote add origin {GitHub_Page Repo}
$ git push -u origin master
```

## 4. posting

```bash
$ mkdir _posts
$ cd _posts

# YYYY-MM-DD-title.md
$ touch 2023-01-08-first-posting.md 
$ vi 2023-01-08-first-posting.md

$ bundle exec jekyll serve
```

## 5. advanced settings
### `_config.yml`
* * *
페이지에 기본 설정을 해줍시다
  ```bash
  #1. 기본 구성 Permalink
  locale                   : "en-US"
  title                    : "Amazing Site"
  title_separator          : "-"
  name                     : "멋있게 성장중인 개발자"
  url                      : "https://chaneeh.github.io"

  #2. 저자 소개 Permalink
  author:
    name             : "멋있게 성장중인 개발자"
    bio              : "I am an **amazing** person. \n  I am a **growing** person."
    location         : "South Korea"
    email            :
    links:
      - label: "Email"
        icon: "fas fa-fw fa-envelope-square"
        # url: "mailto:your.name@email.com"
  ```

### `_pages/{page_name}.md`
* * *
tag, category 별로 분류해주는 페이지를 만듭시다

  ```bash
  category-archive.md
  tag-archive.md
  404.md
  about.md
  ```

### `_data/navigation.yml`
위에서 생성한 각 page들을 navigation bar를 통해 연결합니다
* * *

  ```bash            
  # main links
  main:
    - title: "home"
      url: https://chaneeh.github.io/
    - title: "Categories"
      url: /categories/
    - title: "Tags"
      url: /tags/
    - title: "About"
      url: /about/
  ```



## 6. reference

- **M1 Mac에서 github.io 블로그 준비하기** 😃
    - [https://choijaegwon.github.io/githubblog/GithubBlog1/](https://choijaegwon.github.io/githubblog/GithubBlog1/)
- **theme and posting** 😃
    - [https://devinlife.com/howto github pages/new-blog-from-template/](https://devinlife.com/howto%20github%20pages/new-blog-from-template/)
- *ruby version*
    - https://github.com/jekyll/jekyll/issues/9233