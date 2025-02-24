## 파일구조 및 사용방법

### 디렉토리 구조

```
├── Edit me to generate
├── _data/
│   └── home.yml
├── include/
│   ├── card_list.html
│   ├── custom_head.html
│   └── ...
├── layout/
│   ├── blog.hmtl
│   ├── default.html
│   ├── home.html
│   └── post.html
├── post/
│   ├── 2023-11-06-....md
│   ├── 2023-11-06-....md
│   └── ...
├── _config.yml
├── 404.html
├── favicon.ico
├── about.md
├── Gemfile
├── Gemfile.lock
├── about.md
├── study.html
└── blog.html
```

---

<br>

### \_layout

layout에서의 최상위 구조는 `default.html`이다.

```html
<!DOCTYPE html>
<html lang="{{ page.lang | default: "en" }}" class="html" data-theme="{{ site.theme_config.appearance | default: "auto" }}">
        {%- include head.html -%}
  <body>
    <main class="page-content" aria-label="Content">
      <div class="w">

        <!-- content here -->
        {{ content }}

      ....

      </div>
    </main>
  </body>
</html>
```

`{{content}}` 라는 템플릿 변수에는 html 상위에 `layout: default`라고 명시한 html의 content가 들어간다.

```html
---
layout: default
---

<!-- 경로 /_layout/study.hmtl -->
<!-- {{content}} 내부에 들어갈 내용을 작성한다.-->

<a href="{{ "/" | relative_url }}">{{ site.theme_config.back_home_text }}</a>

<header>
  <h1>{{ site.theme_config.study.title }}</h1>
</header>

{% include post_list.html %}
```

#### default.html

- 최상위 부모
- 설명
  - 전체 레이아웃 설정
  - `{{content}}` 내부에 페이지의 내용이 들어감

#### home.html

- layout의 `{{content}}` 내부에 설정함
- 사용

```
// 상위에 layout을 명시함
---
layout: default
---
```
---

<br>

### header

url 설정

```yml
// _data.yml
navbar_entries:
  - title: about
    url: about

  - title: blog
    url: blog

  - title: study
    url: study
```

`header`에 직접 마크다운 링크를 두고 싶다면 최상위 경로에 마크다운을 위치시킨다.

`html`을 두고 싶다면 `html`을 생성하고 해당 `html` 명을 `url`에 입력한다.

- `html`을 작성하여 다른 페이지를 생성해준다.

<br>

> 예시) tech 페이지를 추가하고 싶다.

1. 최상위 경로에 `tech.html` 를 추가
2. tech.html에 layout을 명시함 (해당 layout은 \_layout 디렉토리에 위치시킨다.)
   ```html
   --- layout: study ---
   ```
3. `_layout`에 tech.html을 생성

   1. html 내부의 jekyll 변수를 사용한다.
      1. site : Javascript를 비유하면 객체를 선언한 최상위 변수명이다.
      2. theme_config : `_config.yml` 내부에 선언한 속성이다. 해당 속성의 하위 속성을 `.`로 표현한다.
      3. `{% include post_list.html %}` : 다른 html을 import 시킨다.

   ```html
    ---
    layout: default
    ---
    <a href="{{ "/" | relative_url }}">{{ site.theme_config.back_home_text }}</a>

    <header>
      <h1>{{ site.title }}</h1>
    </header>

    {% include post_list.html %}
   ```

<br>

### Blog 작성 순서

### 문법

| 키워드 |               |     |
| ------ | ------------- | --- |
| site   | 최상위 변수명 |     |

### \_config.yml

site.


### \_posts

md 파일에

|            |                   | 데이터 위치                     |
| ---------- | ----------------- | ------------------------------- |
| {{ page }} |                   | page는 markdown 파일명을 의미함 |
|            | {{ page.author }} |                                 |

{{ page.date }}

- \_data
  - home.yml


### jekyll 문법 이해

#### include 

`horizontal_list.html`로 `collection` 변수에 `site.data.home.blog_entries`를 담아 보낸다는 뜻

`horizontal_list.html`은 `_include` 파일에 존재해야 한다. `_include` 디렉토리 내부에서 `horizontal_list.html`를 찾기때문

```
{% include horizontal_list.html collection=site.data.home.blog_entries %}
```

변수명은 다양하게 사용자가 지정할 수 있으며 아래와 같이 다양한 데이터 타입을 담을수 있는듯하다.

```
{% include blog_item.html title='Java'%}
```

blog_item.html의 내부에서는 아래와 같이 변수를 사용할 수 있다.

```
{{ include.title }}
```

해당 값은 변수이므로 for 문, if 문 등 내부에서 사용이 가능하다. (아래는 for문이며, if문도 사용 가능하다.)

```
{% if post.tags contains include.title %}

    //...

{% endif %}
```

더 자세한 사항은 아래 `jekyll include` 문서를 참조

참고 : [https://jekyllrb-ko.github.io/docs/includes/](https://jekyllrb-ko.github.io/docs/includes/)