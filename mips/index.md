---
layout: default
title: All MIPs
nav_order: 2
has_children: true
---

# Midnight Improvement Proposals

Browse all MIPs below. Each proposal goes through a formal review process before implementation.

## MIP List

| MIP | Title | Category | Status |
|:----|:------|:---------|:-------|
{% for mip in site.mips %}{% if mip.title != "All MIPs" %}| [{{ mip.mip_number | default: "TBD" }}]({{ mip.url | relative_url }}) | {{ mip.title }} | {{ mip.category | default: "Core" }} | {{ mip.status | default: "Draft" }} |
{% endif %}{% endfor %}

---

## Submit a New MIP

Ready to propose an improvement? Follow these steps:

1. **Review existing MIPs** to avoid duplicates
2. **Use the template** from [mip-template.md](https://github.com/midnightntwrk/midnight-improvement-proposals/blob/main/mips/mip-template.md)
3. **Submit a PR** to the repository
4. **Engage** with the community during review

[Submit a MIP](https://github.com/midnightntwrk/midnight-improvement-proposals/pulls){: .btn .btn-primary }
