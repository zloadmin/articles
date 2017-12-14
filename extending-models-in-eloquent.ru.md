# Расширение моделей в Eloquent ORM

![](https://habrastorage.org/webt/fc/hn/_v/fchn_v6aqfi1iqi1mph6nz1vcxy.png)

Мы прошли долгий путь, с тех дней когда мы в ручную писали SQL запросы в наших веб приложения. Инструменты, такие как Laravel’ий [Eloquent ORM](https://laravel.com/docs/master/eloquent) позволяют нам работать с базой данных на более высоком уровне, освобождают нас от деталей более низкого уровня - таких как синтаксис запросов и безопасность. 

Когда вы начнете работать с Eloquent, вы неизбежно придете к таким операторам как `where` и `join`. Для более продвинутых есть заготовки запросов (scopes), читатели (accessors), мутаторы (mutators) - предлагающие более выразительные альтернативы, старому способу построения запросов.

Давайте рассмотрим другие альтернативы, которые могут быть использованы как замена часто повторяющемуся оператору `where` и заготовкам запросов (scopes). Эта технология заключается в создание новой модели Eloquent которая будет наследоваться от другой модели. Такая модель будет наследовать весь функционал родительской модели, сохраняя возможность добавлять собственные методы, заготовки запросов (scopes), слушателей (listeners), и т.д. Обычно такое называется "Однотабличное Наследование" (Single Table Inheritance), но я предпочитаю называть это "Модельным Наследованием" (Model Inheritance).

## Пример

Большинство веб приложений имеют концепцию "администратор." Администратор это обычный пользователь с повышенными правами и доступом в служебные части приложения. Для того чтобы отличить обычных пользователей от администраторов мы пишем что то подобное:

    $admins = User::where('is_admin', true)->get();

Когда выражение `where` часто повторяется в вашем приложение, его полезно заменить на локальную заготовку запроса ([local scope](https://laravel.com/docs/master/eloquent#local-scopes)). Внедрив заготовку запроса `isAdmin` в модель `User`, мы сможем писать более выразительный и переиспользуемый код:

    $admins = User::isAdmin()->get();

    // Реализация:
    class User extends Model
    {
        public function scopeIsAdmin($query)
        {
            $query->where('is_admin', true);
        }
    }

Давайте пойдем дальше и используем наследование модели. Наследуясь от модели `User` и добавляя глобальную заготовку запроса, мы достигаем более аккуратного результата чем получали прежде, но сейчас с совершенно новым объектом. Этот объект (`Admin`) может иметь собственные методы, заготовки запросов, и другие функциональные возможности.

    $admins = Admin::all();

    // Реализация:
    class Admin extends User
    {
        protected $table = 'users';

        public static function boot()
        {
            parent::boot();

            static::addGlobalScope(function ($query) {
                $query->where('is_admin', true);
            });
        }
    }

> Примечание: переменная `protected $table = ‘users’` необходима для правильной работы запросов. Eloquent использует имя класса модели для определения имени таблицы. Следовательно Eloquent предполагает что имя таблицы “admins” вместо “users”, что приведет к ошибке `Base table or view not found`.

Теперь когда у вас есть модель `Admin` вам будет проще разделять функциональность с моделью `User`. Например:

#### Нотификации

Простые операции, такие как отправка нотификаций всем администраторам, стала проще с новой моделью `Admin`.

    Notification::send(Admin::all(), NewSignUp($user));

#### Проверка

Всегда когда операции с моделью `User` ограничиваются администратором, нам требуется проверить что пользователь олицетворяет администратора.


    // Проверка
    if ($admin = User::find($id)->is_admin !== true) {
          throw new Exception;
    }

    $admin->impersonate($user);


Так как `Admin`’ая глобальная заготовка запроса ограничивает нас только администраторами, метод `impersonate` можно вызывать сразу для класса `Admin`.

    Admin::findOrFail($id)->impersonate($user);

#### #### Фабрики моделей

Во время тестирования, вам может понадобиться создать модель `User` c привилегиями администратора, используя [фабрику моделей](https://laravel.com/docs/master/database-testing#writing-factories) как в примере ниже.

    $admin = factory(User::class)->create(['is_admin' => true]);

    // // Реализация фабрики пользователя
    $factory->define(User::class, function () {
        return [
            ...
              'is_admin' => false,
        ];
    });

Мы можем улучшить этот код добавив [состояние для фабрики](https://laravel.com/docs/master/database-testing#factory-states) инкапусулировав то - что пользователь является администратором.

    $admin = factory(User::class)->states('admin')->create();

    // Реализация состояния администратора 
    $factory->state(User::class, 'admin', function () {
        return ['is_admin' => true];
    });


Стало несомненно лучше, но мы по прежнему получаем экземпляр модели `User`. Определив новую фабрику для модели `Admin`, мы также получим пользователя с правами администратора, но теперь фабрика будет возвращать экземпляр модели `Admin`.

    $admin = factory(Admin::class)->create();

    // Реализация фабрики администратора
    $factory->define(Admin::class, function () {
        return ['is_admin' => true]
              + factory(User::class)->raw();
    });

## ## Отношения не работают.

Аналогично тому как Eloquent определяет имена таблиц, имена класса моделей используется для для определения внешних ключей и промежуточных таблиц. Следовательно доступ к отношениям из модели `Admin` проблематичен.

    Admin::first()->posts;
    // Бросит исключение: Unknown column 'posts.admin_id'

    // Не рабочая реализация:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function posts() {
              return $this->hasMany(Post::class);
        }
    }

Eloquent не может получить доступ к отношению так как предполагает что каждый экземпляр модели `Post` имеет поле `admin_id` вместо поля `user_id`. Мы можем исправить это передав внешний ключ `user_id` в модели `User`:

    // Рабочая реализация:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function posts() {
              return $this->hasMany(Post::class, 'user_id');
        }
    }

Эта же проблема существует в отношение многие ко многим. Eloquent предполагает что имя промежуточной таблицы соответствует имени текущего класса модели:

    Admin::first()->tags;
    // Бросает исключение: Table 'admin_tag' doesn't exist

    // Не рабочая реализация:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function tags() {
              return $this->belongsToMany(Tag::class);
        }
    ...

Мы так же мы можем решить эту проблему явно указав имя сводной таблицы и имя удаленного ключа:

    // Рабочая реализация:
    class Admin extends User {
        //
    }

    class User extends Model {
        public function tags() {
              return $this->belongsToMany(Tag::class, 'user_tag', 'user_id');
        }
    ...


Несмотря на то что явное определение удаленных ключей и сводных таблиц позволит модели `Admin` получить доступ к отношениям модели `User`, это решение далеко от идеального. Существование этих на вид не нужных определений, не улучшает наш код.

Однако, вы можете создать трейт `HasParentModel` который автоматически решит эту проблему. Данный трейт заменит имя класса модели на имя класса родительской модели. Код трейта [GitHub](https://gist.github.com/calebporzio/a63b165b500d491a0c250eb5853e5d94).

> Можно пойти дальше, и заставить Laravel лучше работать с однотабличным наследованием. Мы создали пакет который упростит создание моделей в вашем Laravel приложения, и готовы выпустить его со дня на день. Следите за нашим [твитером](http://twitter.com/tightenco/) чтобы не пропустить анонс!

Давайте посмотрим на новую модель `Admin` которая использует этот трейт:

    use App\Abilities\HasParentModel;

    class Admin extends User
    {
        use HasParentModel;
          // Больше не нужна переменная: protected $table = 'users'

        public static function boot()
        {
            parent::boot();

            static::addGlobalScope(function ($query) {
                $query->where('is_admin', true);
            });
        }
    }

Сейчас наши отношения модели `User` могут вернуться к тому состоянию когда они полагались на значения по умолчанию.

	// Рабочая реализация:
	class User extends Model
	{
		public function posts() {
			return $this->hasMany(Post::class);
		}
		
		public function tags() {
			return $this->belongsToMany(Tag::class);
		}
	}


Трейт `HasParentModel` очищает нашу модель и дает разработчику понять что что-то особенное происходит внутри нее.


## Наследование моделей

Мы выявили общие характеристики Eloquent модели и сделали их чище, используя их наследование. Эта технология позволяет создавать нам более лучшие имена объектов и инкапсулировать их в нашем приложение. Помните что наследование доступно для всех моделей Eloquent'а, не только для `Users` и `Admins`. Возможности безграничны!

Творите, получайте удовольствие и делитесь полученными знаниями. Поделитесь со мной, как вы используете этот паттерн в ваших проектах! (Твитер [@calebporzio](https://twitter.com/calebporzio) и [@tightenco](https://twitter.com/tightenco))

Удачи!

Источник: [https://tighten.co/blog/extending-models-in-eloquent](https://tighten.co/blog/extending-models-in-eloquent)