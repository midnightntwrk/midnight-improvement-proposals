---
layout: default
title: Home
---

# Midnight Improvement Proposals

A Midnight Improvement Proposal (MIP) is a formalised design document for the Midnight community and the name of the process by which such documents are produced and listed. A MIP provides information or describes a change to the Midnight ecosystem, processes, or environment concisely and in sufficient technical detail.

## MIP Status

| MIP | Title | Status | Category |
| :-: | :--- | :----: | :------- |
{% assign mips = site.pages | sort: "MIP" %}
{% for mip in mips %}
  {% if mip.path contains 'mips/' and mip.MIP %}
| [{{ mip.MIP }}]({{ mip.url | relative_url }}) | [{{ mip.Title }}]({{ mip.url | relative_url }}) | {{ mip.Status }} | {{ mip.Category }} |
  {% endif %}
{% endfor %}

## Contributing

Please review [MIP-1]({{ '/mips/mip-0001-Midnight-Improvement-Proposal-Process.html' | relative_url }}) for the full process.
