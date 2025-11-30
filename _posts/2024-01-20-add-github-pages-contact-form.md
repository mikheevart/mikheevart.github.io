---
title: "Добавление контактной формы на сайт Github Pages"
classes: wide
header:
  teaser: /assets/images/github_pages_572846.jpg
  image: /assets/images/jekyll-github.png
  og_image: /assets/images/github_pages_572846.jpg
category:
 - How To
---

## Что такое страницы Github?

![image-right](/assets/images/github-pages-logo-min.png){: .align-left} Github , конечно, хорошо известен как хранилище исходного кода. Он также поддерживает Github Pages ,
который представляет собой службу хостинга контента, уже находящегося в репозитории на Github. 
Для общедоступных репозиториев и платных аккаунтов Github позволяет разместить собственный сайт через Github Pages. 
Страницы могут быть либо основаны на Jekyll, который используется по умолчанию, либо вы можете предоставить свой 
собственный статический контент, добавив файл .nojekyll .

### Контактная форма на страницах Github

Поскольку Github Pages — это не что иное, как хостинг ваших собственных статических файлов, вы не можете добавить 
контактную форму без внешнего обработчика. Многие пользователи Github Pages используют Un-static Forms в качестве
обработчика, потому что они просты в использовании. Единственное, что вам нужно сделать, это добавить контактную
форму в формате HTML, на которую вы ссылаетесь с других страниц. После этого вы можете передать обработку ваших форм
Un-static Forms. Таким образом, вам не нужен сервер для обработки форм!


### Пример контактной формы на страницах Github

Чтобы включить контактную форму на Github Pages, вы можете выполнить следующие простые шаги:

Шаг 1. Зарегистрируйте форму :point_right: https://forms.un-static.com/register-form

Шаг 2. Возьмите содержимое ниже и поместите его в contact.html (или что-то подобное).

Шаг 3. Замените YOUR_ENDPOINT_REFERENCE в содержании на конечную точку вашей формы.

Шаг 4. Добавьте файл в репозиторий вашего сайта и отправьте его на Github.

```
$ git add contact.html
$ git commit -m 'Add contact form'
$ git push
```
И все готово.. :wink:

Если вы перейдете на URL-адрес своего сайта Github Pages, теперь вы сможете использовать контактную форму и получать любые материалы на свой адрес электронной почты!

Пример исходного кода contact.html
Этот пример не имеет никакого дизайна и основан на Bootstrap 4.

```html
<html>
<head>
  <title>Contact me</title>
</head>

<body>
<form method="post" action="https://forms.un-static.com/forms/YOUR_ENDPOINT_REFERENCE">
  <div class="form-group row">
    <label for="name" class="col-4 col-form-label">Name</label>
    <div class="col-8">
      <div class="input-group">
        <div class="input-group-addon">
          <i class="fa fa-user"></i>
        </div>
        <input id="name" name="name" placeholder="Please enter your name" type="text" required="required" class="form-control">
      </div>
    </div>
  </div>
  <div class="form-group row">
    <label for="email" class="col-4 col-form-label">E-mail address</label>
    <div class="col-8">
      <div class="input-group">
        <div class="input-group-addon">
          <i class="fa fa-envelope"></i>
        </div>
        <input id="email" name="email" placeholder="Your e-mail address" type="text" required="required" class="form-control">
      </div>
    </div>
  </div>
  <div class="form-group row">
    <label for="message" class="col-4 col-form-label">Message</label>
    <div class="col-8">
      <textarea id="message" name="message" cols="40" rows="10" required="required" class="form-control"></textarea>
    </div>
  </div>
  <div class="form-group row">
    <div class="offset-4 col-8">
      <button name="submit" type="submit" class="btn btn-primary">Send</button>
    </div>
  </div>
</form>
<div align="center">
  <p><small>(Powered by <a rel="nofollow" href="Un-static Forms">Un-static Forms</a>)</small></p>
</div>
</body>
</html>
```
