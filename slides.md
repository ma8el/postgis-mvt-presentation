---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://images.unsplash.com/photo-1489702932289-406b7782113c?ixlib=rb-4.0.3&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2072&q=80
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
#info: |
#  ## Slidev Starter Template
#  Presentation slides for developers.
#
#  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
css: unocss
---

# Mapbox Vector Tiles with PostGIS

Easy Tile Server on your PostgreSQL database

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/ma8el/postgis-mvt" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

---

# What are Mapbox Vector Tiles?

MVT are ...

- 📝 **Fillme** - Fillme
- 🎨 **Fillme** - Fillme

<br>
<br>

Read more about [PostGIS](https://postgis.net/)

---
layout: image-right
image: https://source.unsplash.com/collection/94734566/1920x1080
---

# Code

Use code snippets and get the highlighting directly![^1]

```ts {all}
interface User {
  id: number
  firstName: string
  lastName: string
  role: string
}

function updateUser(id: number, update: User) {
  const user = getUser(id)
  const newUser = { ...user, ...update }
  saveUser(id, newUser)
}
```

[^1]: [Learn More](https://sli.dev/guide/syntax.html#line-highlighting)

---

# In action

<div grid="~ cols-2 gap-4">
<div>

See the vector tiles in action

</div>
<div>
<Map />
</div>
</div>

---
layout: center
class: text-center
---

# Learn More

[Documentations](https://postgis.net/documentation/) · [GitHub](https://github.com/ma8el/postgis-mvt) · [Showcases](https://google.com)
