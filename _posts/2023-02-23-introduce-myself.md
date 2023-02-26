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
last_modified_at: 2023-02-23T13:06:00+09:00
---

github를 이용하여 개인 홈페이지를 만드는 과정 입니다.

개인 홈페이지를 처음 만드는 분들에게 도움이 되고자 정리합니다!

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

```bash
$ gem install jekyll bundler
$ jekyll -v
$ bundle exec jekyll -v # jekyll 3.2.1

$ bundle add webrick
$ bundle install

$ bundle exec jekyll serve
```

## 3. choose theme & create repo

```bash
$ git clone https://github.com/mmistakes/minimal-mistakes.git
$ mv jekyll/minimal-mistakes/ {GitHub_Page Dir}
$ cd {GitHub_Page Dir}

$ git remote remove origin
$ git remote add origin {GitHub_Page Repo}
$ git push -u origin master
```

## 4. initial setting

```bash
# remove unnecessary files
$ rm .editorconfig
$ rm .gitattributes
$ rm .github
$ rm -r /docs
$ rm -r /test
$ rm CHANGELOG.md
$ rm README.md
$ rm screenshot-layouts.png
$ rm screenshot.png
```

## 5. posting

```bash
$ mkdir _posts
$ cd _posts

$ touch 2023-01-08-first-posting.md # YYYY-MM-DD-title.md
$ vi 2023-01-08-first-posting.md

# local hosting
$ bundle exec jekyll serve
```

## 6. advanced settings
   - ### _config.yml
    
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

   - ### _pages/{page_name}.md
    
        ```bash
        category-archive.md
        tag-archive.md
        404.md
        about.md
        ```

   - ### _data/navigation.yml

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



## 7. reference websites

- **M1 Mac에서 github.io 블로그 준비하기** 😃
    - [https://choijaegwon.github.io/githubblog/GithubBlog1/](https://choijaegwon.github.io/githubblog/GithubBlog1/)
- **theme and posting** 😃
    - [https://devinlife.com/howto github pages/new-blog-from-template/](https://devinlife.com/howto%20github%20pages/new-blog-from-template/)
- *ruby version*
    - https://github.com/jekyll/jekyll/issues/9233