---
layout: post
title: A cheet sheet for building homepage based on al-folio
date: 2024-09-03 22:45:00
description: Include descriptions of some changeable parameters which aren't mentioned in the official repository README
tags: homepage, jekyll, css
categories: cheetsheet
---

Building one's own academic homepage is interesting and useful (e.g. when applying for a graduate research assistant position) and [al-folio](https://github.com/alshedivat/al-folio) is a well-known, easy-to-use and elegant homepage template. In this blog post I will briefly describe some things that are not covered in detail in the official documentation but may be helpful, such as changing the icon size and the image rendering size in the profile, and how to change the teaching module to a project-like format.

### Change the size of blog tag and category icons

The default icon size may not be suitable for all application scenarios, so you may need to adjust the icon rendering size.

<div class="row mt-3">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/icons_example.png" class="img-fluid rounded z-depth-1" zoomable=true %}
    </div>
</div>

The icon rendering of the blog interface is controlled by `h1` in `.header-bar`. If you need to change it, open `_sass/_base.scss` and find the following code

```css
.header-bar {
  border-bottom: 1px solid var(--global-divider-color);
  text-align: center;
  padding-top: 2rem;
  padding-bottom: 3rem;

  h1 {
    color: var(--global-theme-color);
    font-size: 3rem;
  }
}
```

Then change the `font-size` in `h1`. Note that `rem` is different from `px` which is based on certain pixels. It is a unit that changes based on the size of the root element(such as HTML default font size). If you need to control the size accurately to the pixel, you can set it to a value in `px`. Here, recommends setting it to `rem` for the stability of the interface.

### Change picture size in `peoples`

When I modified the content in the people column, I found that the original image rendering size was too large, which caused the interface to lose its compactness (it varies from person to person), so I modified the size of the image rendering here.

The format in `peoples` is affected by `_base.css` and `_layout.css`. Here, the file path of the image rendering size is changed to `_sass/_layout.css` and you can focus on following code.

```css
// Profile
.profile {
  img {
    width: 60%;
  }
}
```

Therefore, change the width here.

### build teaching column

The official documentation only mentions that `teaching` can be changed from markdown to a column like `projects`, but it does not clearly point out the method. The author provides a cheetsheet here for easy configuration.

Firstly, add `teaching` to `_config.yml` as following

```yaml
collections:
  news:
    defaults:
      layout: post
    output: true
    permalink: /:collection/:title/
  projects:
    output: true
    permalink: /:collection/:title/
  teaching:
    output: true
    permalink: /:collection/:title/
```

Then write the following code into `_page/_teaching.md`

```markdown
---
layout: page
permalink: /teaching/
title: this is an example
description: this is an example
nav: true
nav_order: 6
---
<!-- pages/teaching.md -->
<div class="teaching">
{% if site.enable_teaching_categories and page.display_categories %}
  <!-- Display categorized teaching -->
  {% for category in page.display_categories %}
  <a id="{{ category }}" href=".#{{ category }}">
    <h2 class="category">{{ category }}</h2>
  </a>
  {% assign categorized_teaching = site.teaching | where: "category", category %}
  {% assign sorted_teaching = categorized_teaching | sort: "importance" %}
  <!-- Generate cards for each teaching -->
  {% if page.horizontal %}
  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
    {% for teaching in sorted_teaching %}
      {% include teaching_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for teaching in sorted_teaching %}
      {% include teaching.liquid %}
    {% endfor %}
  </div>
  {% endif %}
  {% endfor %}

{% else %}

<!-- Display teaching without categories -->

{% assign sorted_teaching = site.teaching | sort: "importance" %}

  <!-- Generate cards for each teaching -->

{% if page.horizontal %}

  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
    {% for teaching in sorted_teaching %}
      {% include teaching_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for teaching in sorted_teaching %}
      {% include teaching.liquid %}
    {% endfor %}
  </div>
  {% endif %}
{% endif %}
</div>

```

Next, in order for Jekyll to render successfully, we also need to add the liquid file in `_includes`.

teaching.liquid
```html
<div class="col">
  <a href="{% if teaching.redirect %}{{ teaching.redirect }}{% else %}{{ teaching.url | relative_url }}{% endif %}">
    <div class="card h-100 hoverable">
      {% if teaching.img %}
        {%
          include figure.liquid
          loading="eager"
          path=teaching.img
          sizes = "250px"
          alt="teaching thumbnail"
          class="card-img-top"
        %}
      {% endif %}
      <div class="card-body">
        <h2 class="card-title">{{ teaching.title }}</h2>
        <p class="card-text">{{ teaching.description }}</p>
        <div class="row ml-1 mr-1 p-0">
          {% if teaching.github %}
            <div class="github-icon">
              <div class="icon" data-toggle="tooltip" title="Code Repository">
                <a href="{{ teaching.github }}"><i class="fa-brands fa-github gh-icon"></i></a>
              </div>
              {% if teaching.github_stars %}
                <span class="stars" data-toggle="tooltip" title="GitHub Stars">
                  <i class="fa-solid fa-star"></i>
                  <span id="{{ teaching.github_stars }}-stars"></span>
                </span>
              {% endif %}
            </div>
          {% endif %}
        </div>
      </div>
    </div>
  </a>
</div>

```

teaching_horizontal.liquid
```html
<div class="col mb-4">
  <a href="{% if teaching.redirect %}{{ teaching.redirect }}{% else %}{{ teaching.url | relative_url }}{% endif %}">
    <div class="card h-100 hoverable">
      <div class="row no-gutters">
        {% if teaching.img %}
          <div class="col-md-6">
            {% include figure.liquid loading="eager" path=teaching.img sizes="(min-width: 768px) 156px, 50vw" alt="teaching thumbnail" class="card-img" %}
          </div>
        {% endif %}
        <div class="{% if teaching.img %}col-md-6{% else %}col-md-12{% endif %}">
          <div class="card-body">
            <h3 class="card-title">{{ teaching.title }}</h3>
            <p class="card-text">{{ teaching.description }}</p>
            <div class="row ml-1 mr-1 p-0">
              {% if teaching.github %}
                <div class="github-icon">
                  <div class="icon" data-toggle="tooltip" title="Code Repository">
                    <a href="{{ teaching.github }}"><i class="fa-brands fa-github gh-icon"></i></a>
                  </div>
                  {% if teaching.github_stars %}
                    <span class="stars" data-toggle="tooltip" title="GitHub Stars">
                      <i class="fa-solid fa-star"></i>
                      <span id="{{ teaching.github_stars }}-stars"></span>
                    </span>
                  {% endif %}
                </div>
              {% endif %}
            </div>
          </div>
        </div>
      </div>
    </div>
  </a>
</div>

```

Since `teaching` is modified based on the `projects` type, please find a method yourself if other types are required.

The above is the full content of this blog. If I find any problems later, I will continue to update this article.