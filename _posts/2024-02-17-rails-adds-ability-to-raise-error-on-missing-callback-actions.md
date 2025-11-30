---
title: В Rails 7.1 добавлена ​​возможность выдавать ошибки при отсутствии действий обратного вызова.
description: Контроллеры Rails имеют обратные вызовы, позволяющие запускать код в определенных точках жизненного цикла запроса.
---
Контроллеры Rails имеют обратные вызовы, позволяющие запускать код в определенных точках жизненного цикла запроса. Вы можете использовать параметры only или except, чтобы указать, для каких действий должен выполняться обратный вызов.
```ruby
class UsersController < ApplicationController
  before_action :set_user, except: [:index, :new, :create]
  after_action :notify_admins, only: [:create]
end
```
В предыдущих версиях Rails параметры only и exceptв обратных вызовах контроллера не проверяли, существуют ли действия, на которые они ссылаются. Это означало, что вы могли допустить опечатку в названии действия, и обратный вызов не будет выполнен, хотя он и должен был выполняться.

onlyВ приведенном выше примере, если в опции обратного вызова допущена опечатка after_action, notify_adminsметод для действия никогда не будет выполнен create. Администраторы вашей системы никогда не будут получать никаких уведомлений о новых регистрациях на вашей платформе.
```ruby
class UsersController < ApplicationController
  before_action :set_user, except: [:index, :new, :create]
  after_action :notify_admins, only: [:crete] # typo crete => create
end
```
Рассмотрим другой сценарий, если вы выполните рефакторинг своей кодовой базы и измените имя действия контроллера, на которое ссылается обратный вызов. Вы можете случайно внедрить ошибку
```ruby
class UsersController < ApplicationController
  before_action :authorize_user, only: [:dashboard]

  # many lines of code of other actions

  # method name changed from +dashboard+ to +admin_dashboard+
  def admin_dashboard
  end
end
```
В приведенном выше примере обычный пользователь вашей системы может использовать ошибку в вашем коде для доступа к панели администратора и взломать важные данные.
### До Rails 7.1
До Rails 7.1 существовало два способа проверки этих проблем.

* Первым способом было охватить все сценарии тестирования контроллера. Это обеспечит проверку всех возможных путей через контроллер, включая те, которые связаны с обратными вызовами.

* Вторым способом было написать скрипт, проверяющий недостающие действия, добавленные в only/exceptопции. Этот сценарий будет сканировать код контроллера на предмет любых обратных вызовов, которые ссылаются на несуществующие действия.

### В Рельсах 7.1

Rails 7.1 позволяет настроить контроллеры так, чтобы они выдавали ошибку, когда в опциях only/Exception в обратном вызове отсутствуют действия .

Когда before_action :callback, only: :action_nameобъявлено в контроллере, который не отвечает на action_name, оно вызывает исключение во время запроса. Это мера безопасности, гарантирующая, что опечатки или забывчивость не помешают выполнению важного обратного вызова.

Эта функция включена для новых приложений с Rails 7.1 по умолчанию. Конфигурация находится trueв Rails config/environments/development.rb и config/environments/test.rbфайлах. Чтобы отключить это поведение, вы должны установить параметр falseв конфигурации вашего приложения.

```ruby
config.action_controller.raise_on_missing_callback_actions = false
```
В приведенном ниже варианте UsersControllerесть опечатка after_action :notify_admins only. Когда приложение запрашивает какое-либо действие в контроллере, оно вызывает следующую ошибку.
```ruby
class UsersController < ApplicationController
  before_action :set_user, except: [:index, :new, :create]
  after_action :notify_admins, only: [:crete] # typo crete => create
end

# Request for UsersController#index action will raise the error

The crete action could not be found for the :notify_admins
callback on UsersController,
but it is listed in the controller's
:only option.

Raising for missing callback actions is a new default in Rails 7.1,
if you'd
like to turn this off you can delete the option from the environment configurations
or set `config.action_controller.raise_on_missing_callback_actions` to `false`.
```

Добавление неопределенного действия в параметры обратного вызова — это ошибка, которую следует выявить во время разработки. Эта функция полезна при рефакторинге кода или при обнаружении опечаток при кодировании.

#### Когда следует избегать этой функции?

Если в вашем приложении есть проблемы с общим контроллером, возможно, вам придется отключить эту функцию.

```ruby
module AdminNotification
  extend ActiveSupport::Concern

  included do
    before_action :notify_admins, only: [:create, :destroy]
  end
end

class UsersController < ApplicationController
  include AdminNotification

  def create
  end
end

class Admins::UsersController < ApplicationController
  include AdminNotification

  def destroy
  end
end
```
У концерна AdminNotificationесть файл before_action, который включен в UsersController и Admins::UsersController. Запрос любого действия в файле UsersControllerприведет к ошибке из-за отсутствующего destroyдействия. Аналогично, ошибка отсутствия createдействия возникает при запросе любого действия в файле Admins::UsersController. Отключение этой функции имеет больше смысла в подобных сценариях.

