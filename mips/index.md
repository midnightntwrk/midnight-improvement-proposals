---
title: MIPs
layout: default
nav_order: 2
permalink: /mips/
---

# Midnight Improvement Proposals (MIPs)

Concrete, technically-precise proposals to change the Midnight ecosystem. See
[MIP-0001]({{ "/mips/mip-0001-mip-process/" | relative_url }}) for the full process and
[`mip-template.md`](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-template.md)
for the template.

| # | Title | Status | Category |
|---|-------|--------|----------|
{% assign mips = site.pages | where_exp: "p", "p.path contains 'mips/mip-'" | sort: "path" -%}
{% for p in mips -%}
{% unless p.path contains 'template' -%}
| {{ p.MIP }} | [{{ p.Title }}]({{ p.url | relative_url }}) | {{ p.Status }} | {{ p.Category }} |
{% endunless -%}
{% endfor %}
