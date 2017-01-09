---
layout: post
title:  "Обработка кастомных ошибок в Rails и переводы"
date:   2017-01-08 14:05:00 +0600
categories: Rails Ruby
excerpt: "Считается, что показатель продуктивного дня — крепкий сон. Отлично будут спать люди, которые весь день стучали молотком по земле, если конечно у них нет болезни Паркинсона. Правда толку от стучания молотком не много, разве, что рука станет крепче. И вот стучит работяга, стучит, весь день стучит, приходит домой, а сил хватает только на поесть и дойти до кровати. К сожалению большинство профессий в Казахстане примерно об этом. У нас все работают как крабы на галерах, уста..."
---
Любое веб-приложение несет определенную модель поведения, а у каждой модели есть свои правила. Правила нельзя нарушать, а при попытке это сделать необходимо сообщить об ошибке. В данном посте хочу поделиться способом обработки ошибок не нарушая пресловутый «Rails Way». Все это заправим переводами и выводом сообщений пользователю. Попробуем перейти от самой простой модели к финальной, на мой взгляд, наиболее правильной. Этого будет достаточно для среднего размера приложений с большим количеством кастомных ошибок. 

Давайте запретим удаление пользователя, если он является администратором.

Многие приложения на Rails не реализуют кастомные ошибки, предпочитая использовать StandardError. Код может выгледь вот так:

{% highlight ruby %}
# models/user.rb
def destroy
  raise StandardError, 'Oops! Admin is immortal!' if admin?
  super
end

# controllers/users_controller.rb
def destroy
  # ...
  rescue StandardError => e
  redirect_to some_path, error: e.msg
end
{% endhighlight %}

Все очень просто, но такая реализация несет ряд проблем. Вместе с нашей ошибкой мы можем показать пользователю и системную, включая ошибки на стороне БД. Это будет похоже на ситуацию, когда в PHP трейс ошибки вываливается на страницу. Здесь происходит похожая ситуация. Попробуем исправить.

Создадим папку `errors` в директории `app`, где будут храниться ошибки нашего приложения. Скажем Rails подгружать папку в проект. Для этого добавим строчку кода в `config/application.rb`. 

{% highlight ruby %}
class Application < Rails::Application
  # ...
  config.autoload_paths << "#{config.root}/app/errors"
  # ...
end
{% endhighlight %}

Теперь можно создать наш первый класс ошибки. Унаследуем его от `StandardError`:

{% highlight ruby %}
# errors/admin_destroy_error.rb
class AdminDestroyError < StandardError
  def initialize
    super(msg = 'Oops! Admin is immortal!')
  end
end
{% endhighlight %}

Перепишем наши модель и контроллер:

{% highlight ruby %}
# models/user.rb
def destroy
  raise AdminDestroyError if admin?
  super
end

# controllers/users_controller.rb
def destroy
  # ...
  rescue AdminDestroyError => e
  redirect_to some_path, error: e.msg
end
{% endhighlight %}

Все стало значительно красивее. Ошибку можно переиспользовать, вся логика в отдельном классе, и самое главное мы делаем `rescue` только нужной нам ошибки. Очень часто этого достаточно, но приложение растет и при удалении может происходить множество проверок. В итоге наш контроллер превратится в кашу:

{% highlight ruby %}
# controllers/users_controller.rb
def destroy
  # ...
  rescue AdminDestroyError, AnotherError, ..., AnotherOtherError => e
  redirect_to some_path, error: e.msg
end
{% endhighlight %}

Самое простое решение этой проблемы — создание базового класса для всех кастомных ошибок. Добавим базовый класс и перепишем наш `AdminDestroyError`.

{% highlight ruby %}
# errors/business_logic_error.rb
class BusinessLogicError < StandardError
  def initialize(msg = I18n.t("errors.#{self.to_s.underscore}"))
    super(msg)
  end
end

# errors/admin_destroy_error.rb
class AdminDestroyError < BusinessLogicError; end
{% endhighlight %}

Тут мы немного схитрили и использовали гем интернационализации i18n включенный по умолчанию в Rails. Конструктор передаст сообщение ошибки по названию класса. Все, что у нас должно быть, это файл с переводами в `config/locales/`.

{% highlight yaml %}
en:
  errors:
    admin_destroy_error: 'Oops! Admin is immortal!'
ru:
  errors:
    admin_destroy_error: 'Упс! Админ пуленепробиваем!'
{% endhighlight %}

Теперь все наши ошибки занимают файлик с одной строчкой (а можно все их закинуть в один файл), работают с переводами и выглядят вот так:

{% highlight ruby %}
# errors/another_error.rb
class AnotherError < BusinessLogicError; end

# errors/another_other_error.rb
class AnotherOtherError < BusinessLogicError; end
{% endhighlight %}

Перепишем наш контроллер, чтобы обрабатывать все эти ошибки, не задевая системных.

{% highlight ruby %}
# controllers/users_controller.rb
def destroy
  # ...
  rescue BusinessLogicError => e
  redirect_to some_path, error: e.msg
end
{% endhighlight %}

Таким образом все ошибки унаследованные от `BusinessLogicError` красиво обрабатываются и легко масштабируются. Можно создать несколько базовых классов ошибок пользовательского приложения, например `ApiError`, `BackgroundJobError`.

Можем пойти дальше и переиспользовать один класс ошибок, если все они вертятся вокруг одной сущности. Например наша модель `User` пока имеет одну ошибку `AdminDestroyError`, а в конце концов может иметь 10, 20 и больше ошибок. В таком случае навигация по классам ошибок может стать проблемой, названия классов будут длинными и не понятно, что и к чему относится. В таких случаях можно создать один класс ошибки для модели `User` и передавать уточненное наименование в качестве аргумента. Еще один плюс подхода в том, что наименования ошибок становятся декларативными (мы уже не говорим «Ошибка удаления админа», мы говорим «Ошибка пользователя — Администратор не может быть удален», ).

{% highlight ruby %}
# errors/user_error.rb
class UserError < BusinessLogicError
  def initialize(msg)
      super(I18n.t("errors.user_errors.#{msg}"))
    end
end
{% endhighlight %}

{% highlight yaml %}
en:
  errors:
    user_errors:
      admin_can_not_be_destroyed: 'Oops! Admin is immortal!'
      user_is_blocked: 'Sorry! User is blocked'
      avatar_is_missing: 'Avatar required to continiue, but missing'
{% endhighlight %}

В итоге вызов ошибок будет выглядеть вот так.


{% highlight ruby %}
# models/user.rb
def destroy
  raise UserError, 'admin_can_not_be_destroyed' if admin?
  raise UserError, 'user_is_blocked' if blocked?
  super
end
{% endhighlight %}

В данной статье мы не нарушили принцип «Rails Way» и использовали стандартные инструменты, предоставляемые Rails. Данный подход отлично подходит для приложений малого и среднего размера. В следующих статьях мы поговорим о работе с ошибками в приложениях использующих event-driven архитектуру на примере гема `Whisper`, а также вынесем из `ActiveRecord` обработку ошибок валидации в отельные классы с помощью `reform`. 
