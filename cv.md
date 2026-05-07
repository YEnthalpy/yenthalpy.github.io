---
layout: page
title: CV
permalink: /cv/
---

<div class="cv-viewer">
  <object data="{{ '/assets/cv.pdf' | relative_url }}" type="application/pdf">
    <p>
      Your browser cannot display the CV here.
      <a href="{{ '/assets/cv.pdf' | relative_url }}">Open the PDF instead.</a>
    </p>
  </object>
</div>

<style>
  .cv-viewer object {
    width: 100%;
    height: 85vh;
    min-height: 720px;
  }
</style>
