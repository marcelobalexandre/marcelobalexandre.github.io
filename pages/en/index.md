---
layout: default
locale: en
header_id: home
title: Home
permalink: /
---

<div class="home">
  <section class="about">
    <div class="avatar">
      <img src="https://source.unsplash.com/featured?avatar" alt="Avatar for Marcelo Alexandre" />
    </div>

    <div class="info">
      <h1>Hello!</h1>
      <p>My name is Marcelo Alexandre and I'm a software engineer.</p>
      {%- include social.html -%}
    </div>
  </section>

  <section class="posts last-posts">
    <h2>Last posts</h2>

    <div class="last-posts">
      {%- assign posts = site.posts | where: 'locale', page.locale -%}
      {%- for post in posts limit: 3 -%}
        <div class="post-column">
          <article>
            <a href="{{ post.url | relative_url }}">
              {% if post.image %}
                <div class="image" style="background-image: url({{ post.image }});"></div>
              {% endif %}

              <div class="detail">
                <h2>{{ post.title | escape }}</h2>

                {{ post.excerpt }}

                <span class="date">
                  {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
                  {{ post.date | date: date_format }}
                </span>
              </div>
            </a>
          </article>
        </div>
      {%- endfor -%}
    </div>

    <a href="{{ site.blog_path[page.locale] | prepend: site.baseurl }}" class="secondary-button more-button">
      More
    </a>
  </section>
</div>
