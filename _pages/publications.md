---
layout: page
permalink: /publications/
title: publications
description: Selected publications. For the complete publication list, please visit my Google Scholar profile.
nav: true
nav_order: 1
---

<!-- _pages/publications.md -->

<!-- Bibsearch Feature -->

{% include bib_search.liquid %}

<div class="publications">

Selected publications. For the complete publication list, please visit my [Google Scholar profile](https://scholar.google.com/citations?user=QPEhFegAAAAJ).

{% bibliography --query @*[selected=true] %}

</div>
