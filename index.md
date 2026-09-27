---
# https://vitepress.dev/reference/default-theme-home-page
layout: home

hero:
  name: "Notes"
  # text: "It's happening, right here and now"
  tagline: It's happening, right here and now
  actions:
    - theme: brand
      text: Begin
      link: /README.md
---
<style>
.love-icon {
  position: absolute;
  right: 20px;
  top: -300px;
  width: 500px;
  height: 500px;
}

.love-icon img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

@media (max-width: 1100px) {
  .love-icon {
    position: static !important;
    width: 300px;
    height: 300px;
    margin: 0 auto;
  }
}
</style>

<div class="love-icon">
  <img src="/love.png" />
</div>

