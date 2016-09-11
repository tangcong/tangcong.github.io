---
layout: home
---

<div class="index-content coding">
  <div class="section">

    <ul class="artical-list">
      {% for post in site.categories.coding %}
      <li>
        <div class="table-article">
          <div class="col-title">
            <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
          </div>
          <div class="col-date">
            <p class="entry-date">{{ post.date|date:"%Y-%m-%d" }}</p>
          </div>
        </div>
        <div class="title-desc">{{ post.description }}</div>
      </li>
      {% endfor %}
    </ul>
  </div>
  <div class="aside">
  </div>

</div>
