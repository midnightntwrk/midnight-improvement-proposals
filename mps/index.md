---
title: MPS
layout: default
nav_order: 3
permalink: /mps/
---

# Midnight Problem Statements (MPS)

Solution-agnostic descriptions of friction, gaps, or opportunities in the ecosystem. MPS
documents define *what* needs solving; MIPs propose *how*. See
[MPS-0001]({{ "/mps/mps-0001-mps-process/" | relative_url }}) for the full process and
[`mps-template.md`](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mps/mps-template.md)
for the template.

| # | Title | Status | Category |
|---|-------|--------|----------|
{% assign mpss = site.pages | where_exp: "p", "p.path contains 'mps/mps-'" | sort: "path" -%}
{% for p in mpss -%}
{% unless p.path contains 'template' -%}
| {{ p.MPS }} | [{{ p.Title }}]({{ p.url | relative_url }}) | {{ p.Status }} | {{ p.Category }} |
{% endunless -%}
{% endfor %}
