---
layout: page
title: 项目
permalink: /zh/projects/
lang: zh
alt_url: /projects/
description: 2026 年在天轶人形机器人上的真机工作。每个项目先讲问题，再讲关键洞察和证据。
nav: true
nav_order: 3
---

{% assign zh_projects = site.pages | where: "zh_project", true | sort: "importance" %}

<div class="projects">
  <div class="container">
    <div class="row row-cols-1 row-cols-md-2">
      {% for project in zh_projects %}
        <div class="col mb-4">
          <a href="{{ project.url | relative_url }}">
            <div class="card h-100 hoverable">
              <div class="row no-gutters">
                <div class="col-md-6">
                  {% include figure.liquid loading="eager" path=project.img sizes="(min-width: 768px) 156px, 50vw" alt="project thumbnail" class="card-img" %}
                </div>
                <div class="col-md-6">
                  <div class="card-body">
                    <h3 class="card-title">{{ project.title }}</h3>
                    <p class="card-text">{{ project.description }}</p>
                  </div>
                </div>
              </div>
            </div>
          </a>
        </div>
      {% endfor %}
    </div>
  </div>
</div>
