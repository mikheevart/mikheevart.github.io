---
layout: default
title: Обо мне
permalink: /about/
weight: 3
---
<br>
<br>
Приветствую, меня зовут **{{ site.author.name }}** :wave:,<br> я **веб-разработчик** широкого профиля, работаю с проектами разной сложности (от одностраничных сайтов до CRM систем).

По большей части работал с Ruby on Rails, Django и Wagtail, также работал с другими популярыми CMS системами. Имею опыт :briefcase: работы фрилансером, что помогло мне научиться лучше понимать разные требования заказчика под любые нестандартные задачи.

:envelope: E-mail: [mikheevartist@yandex.ru](mailto:mikheevartist@yandex.ru)

:iphone: Телефон: [+7(963)947-50-34](tel:89639475034)

<hr>

<div class="row">
{% include about/skills.html title="Programming Skills" source=site.data.programming-skills %}
{% include about/skills.html title="Other Skills" source=site.data.other-skills %}
</div>

<div class="row">
{% include about/timeline.html %}
</div>
