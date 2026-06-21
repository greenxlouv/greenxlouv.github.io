# greenxlouv.github.io 세팅 가이드

이 폴더는 GitHub Pages(Jekyll) 블로그 사이트 골격이다. 아래 순서대로 따라 하면 바로 배포된다.

## 1. 레포 생성

GitHub에서 정확히 이 이름으로 새 레포를 만든다 (이 이름이어야 자동으로 사이트가 생성됨):

```
greenxlouv.github.io
```

## 2. 이 폴더 내용 push

```bash
cd 이폴더경로
git init
git remote add origin https://github.com/greenxlouv/greenxlouv.github.io.git
git add .
git commit -m "feat: initial Jekyll blog setup with first post"
git branch -M main
git push -u origin main
```

## 3. GitHub Pages 활성화

1. 레포 페이지 → **Settings** → **Pages**
2. **Source**를 `Deploy from a branch`로, 브랜치는 `main` / `/(root)`로 설정
3. 저장하면 몇 분 안에 `https://greenxlouv.github.io`에서 확인 가능

## 폴더 구조

```
.
├── _config.yml          ← 사이트 설정 (제목, 테마, permalink 등)
├── Gemfile               ← GitHub Pages가 요구하는 Jekyll 의존성 명시
├── index.md              ← 홈페이지 (포스트 목록 자동 출력)
└── _posts/
    └── 2026-06-22-syntree-kv-ast-based-kv-cache-reuse.md   ← 첫 포스트
```

## 새 글 추가하는 법

`_posts/` 폴더에 `YYYY-MM-DD-제목.md` 형식으로 파일을 추가하고, 파일 맨 위에 아래 frontmatter를 넣으면 된다.

```yaml
---
layout: post
title: "글 제목"
date: 2026-06-22
categories: [카테고리1, 카테고리2]
tags: [태그1, 태그2]
---
```

## 테마 바꾸고 싶으면

`_config.yml`의 `theme:` 값을 바꾸면 된다. GitHub Pages가 기본 지원하는 테마 목록:
`jekyll-theme-minimal`, `jekyll-theme-cayman`, `jekyll-theme-architect`, `jekyll-theme-slate`, `jekyll-theme-dinky` 등.

## 로컬에서 미리보기 (선택)

Ruby와 Bundler가 설치되어 있다면:

```bash
bundle install
bundle exec jekyll serve
```

`http://localhost:4000`에서 확인 가능하다.
