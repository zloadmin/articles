20 трюков Eloquent ORM
===================================

Eloquent ORM кажется простой, но под капотом существует много полускрытых функций и менее известных способов. В этой статье я покажу вам несколько трюков.

### 1\. Инкрементация и декрементация

Вместо этого:

    $article = Article::find($article_id);
    $article->read_count++;
    $article->save();
    

Вы можете сделать так:

    $article = Article::find($article_id);
    $article->increment('read_count');
    

Так тоже будет работать:

    Article::find($article_id)->increment('read_count');
    Article::find($article_id)->increment('read_count', 10); // +10
    Product::find($produce_id)->decrement('stock'); // -1
    
* * *

### 2\. X или Y методы

Eloquent есть несколько функций, которые объединяют два метода, например “пожалуйста, сделай X, иначе сделай Y”.

**Пример 1** – `findOrFail()`:

Вместо этого:

    $user = User::find($id);
    if (!$user) { abort (404); }
    

Делаем это:

    $user = User::findOrFail($id);
    

**Пример 2** – `firstOrCreate()`:

Вместо этого:

    $user = User::where('email', $email)->first();
    if (!$user) {
      User::create([
        'email' => $email
      ]);
    }
    

Делаем это:

    $user = User::firstOrCreate(['email' => $email]);
    

* * *

### 3\. Метод boot() модели

В модели Eloquent есть волшебный метод `boot()`, где вы можете переопределить поведение по умолчанию:

    class User extends Model
    {
        public static function boot()
        {
            parent::boot();
            static::updating(function($model)
            {
                // выполнить какую-нибудь логику
                // переопределить какое-нибудь свойство, например $model->something = transform($something);
            });
        }
    }
    

Вероятно, одним из наиболее популярных примеров является установка значения поля на момент создания объекта модели. Предположим, вы хотите сгенерировать поле [UUID](https://github.com/webpatser/laravel-uuid) в этот момент.

    public static function boot()
    {
      parent::boot();
      self::creating(function ($model) {
        $model->uuid = (string)Uuid::generate();
      });
    }
    

* * *


### 4\. Отношения с условием и сортировкой

Это типичный способ определения отношений:

    public function users() {
        return $this->hasMany('App\User');    
    }
    


Но знаете ли вы, что сюда мы можем добавить `where` или `orderBy`?
Например, если вам нужно специальное отношения для некоторых типов пользователей, упорядоченное по электронной почте, вы можете сделать это:

    public function approvedUsers() {
        return $this->hasMany('App\User')->where('approved', 1)->orderBy('email');
    }
    

* * *

### 5\. Свойства модели: timestamps, appends и тд.

Существует несколько «параметров» Eloquent модели в виде свойств класса. Самые популярные из них, вероятно, следующие:

    class User extends Model {
        protected $table = 'users';
        protected $fillable = ['email', 'password']; // какие поля могут быть заполнены выполняя User::create()
        protected $dates = ['created_at', 'deleted_at']; // какие поля будут типа Carbon
        protected $appends = ['field1', 'field2']; // доп значения возвращаемые в JSON
    }
    

Но подождите, есть еще:

    protected $primaryKey = 'uuid'; //  не должно быть "id"
    public $incrementing = false; // и не должно быть автоинкрементом
    protected $perPage = 25; // Да, вы можете переопределить число записей пагинации (по умолчанию 15)
    const CREATED_AT = 'created_at';
    const UPDATED_AT = 'updated_at'; // Да, даже эти названия также могут быть переопределены
    public $timestamps = false; // или не использоваться совсем
    

И есть еще больше, я перечислил наиболее интересные из них, для более подробной информации ознакомьтесь с кодом по умолчанию [abstract Model class](https://github.com/laravel/framework/blob/5.6/src/Illuminate/Database/Eloquent/Model.php) и посмомотрите все используемые трэйты.

* * *

### 6\. Поиск нескольких записей

Всем известен метод `find ()`, правда?

    $user = User::find(1);
    
Я был удивлен, как мало кто знает о том, что он может принимать несколько IDшников в виде массива:

    $users = User::find([1,2,3]);
    

* * *

### 7\. WhereX

Есть элегантный способ превратить это:

    $users = User::where('approved', 1)->get();
    

В это:

    $users = User::whereApproved(1)->get(); 
    

Да, вы можете изменить имя любого поля и добавить его как суффикс в “where”, и оно будет работать как по волшебству.
Также в Eloquent ORM есть предустановленные методы, связанные с датой и временем:

    User::whereDate('created_at', date('Y-m-d'));
    User::whereDay('created_at', date('d'));
    User::whereMonth('created_at', date('m'));
    User::whereYear('created_at', date('Y'));
    

* * *

### 8\. Сортировка с отношениями

Немного больше чем "трюк". Что делать если у вас есть темы форума, но вы хотите их отсортировать, по их последним **постам**? Довольно популярное требование в форумах с последними обновленными темами вверху, правда?


Сначала опишите отдельную связь для **последнего поста** в теме:

    public function latestPost()
    {
        return $this->hasOne(\App\Post::class)->latest();
    }
    

И после, в вашем контроллере, вы можете выполнить такую "магию":

    $users = Topic::with('latestPost')->get()->sortByDesc('latestPost.created_at');
    

* * *

### 9\. Eloquent::when() – без “if-else”

Многие из нас пишут условные запросы с «if-else», что-то вроде этого:

    if (request('filter_by') == 'likes') {
        $query->where('likes', '>', request('likes_amount', 0));
    }
    if (request('filter_by') == 'date') {
        $query->orderBy('created_at', request('ordering_rule', 'desc'));
    }
    

Но лучший способ - использовать `when()`:

    $query = Author::query();
    $query->when(request('filter_by') == 'likes', function ($q) {
        return $q->where('likes', '>', request('likes_amount', 0));
    });
    $query->when(request('filter_by') == 'date', function ($q) {
        return $q->orderBy('created_at', request('ordering_rule', 'desc'));
    });
    

Этот пример может показаться не короче или элегантнее, но правильнее будет пробрасывать параметры:

    $query = User::query();
    $query->when(request('role', false), function ($q, $role) { 
        return $q->where('role_id', $role);
    });
    $authors = $query->get();
    

* * *

### 10\. Модель по умолчанию для отношений

Допустим, у вас есть пост, принадлежащий автору, и Blade код:

    {{ $post->author->name }}
    

Но что, если автор удален или не установлен по какой-либо причине? Вы получите ошибку на подобии "property of non-object".

Конечно, вы можете предотвратить это следующим образом:

    {{ $post->author->name ?? '' }}
    

Но вы можете сделать это на уровне Eloquent отношений:

    public function author()
    {
        return $this->belongsTo('App\Author')->withDefault();
    }
    

В этом примере отношение `author()` возвращает пустую модель `App\Author`, если автор не прикреплен к посту.

Кроме того, мы можем присвоить значениям свойств по умолчанию для этой модели.

    public function author()
    {
        return $this->belongsTo('App\Author')->withDefault([
            'name' => 'Guest Author'
        ]);
    }
    

* * *


### 11\. Сортировка по преобразователю

Представьте что у вас есть такой преобразователь:

    function getFullNameAttribute()
    {
      return $this->attributes['first_name'] . ' ' . $this->attributes['last_name'];
    }
    

Вам нужно отсортировать записи по полю `full_name`? Такое решение работать не будет:

    $clients = Client::orderBy('full_name')->get(); // не работает
    

Решение довольно простое. Нам нужно отсортировать записи **после** того как мы их получили.

    $clients = Client::get()->sortBy('full_name'); // работает!
    
Обратите внимание что название функций отличается - это не **orderBy**, это **sortBy**.

* * *

### 12\. Сортировка по умолчанию

Что делать, если вы хотите, чтобы `User::all()` всегда сортировался по полю `name`? Вы можете назначить глобальную заготовку (Global Scope). Вернемся к методу `boot ()`, о котором мы уже говорили выше.

    protected static function boot()
    {
        parent::boot();
    
        // Сортировка по полю name в алфавитном порядке
        static::addGlobalScope('order', function (Builder $builder) {
            $builder->orderBy('name', 'asc');
        });
    }
    

* * *

### 13\. Сырые выражения

Иногда нам нужно добавить сырые выражения в наш Eloquent запрос. 
К счастью, для этого есть функции.

    // whereRaw
    $orders = DB::table('orders')
        ->whereRaw('price > IF(state = "TX", ?, 100)', [200])
        ->get();
    
    // havingRaw
    Product::groupBy('category_id')->havingRaw('COUNT(*) > 1')->get();
    
    // orderByRaw
    User::where('created_at', '>', '2016-01-01')
      ->orderByRaw('(updated_at - created_at) desc')
      ->get();
    

* * *

### 14\. Репликация: сделать копию записи

Без глубоких объяснений, вот лучший способ сделать копию записи в базе данных:

    $task = Tasks::find(1);
    $newTask = $task->replicate();
    $newTask->save();
    

* * *

### 15\. Chunk() метод для больших таблиц

Не совсем о Eloquent, это скорее о коллекциях, но все же мощный метод - для обработки больших наборов данных. Вы можете разбить их на кусочки.

Вместо этого:

    $users = User::all();
    foreach ($users as $user) {
        // ...
    

Вы можете сделать это:

    User::chunk(100, function ($users) {
        foreach ($users as $user) {
            // ...
        }
    });
    

* * *

### 16\. Создание дополнительных файлов при создании модели

Мы все знаем Artisan команду:

    php artisan make:model Company
    

Но знаете ли вы, что есть три полезных флага для создания дополнительных файлов модели?

    php artisan make:model Company -mcr
    

*   -m создаст файл миграции (**migration**)
*   -c создаст контроллер (**controller**)
*   -r указывает на то что контроллер должен быть ресурсом (**resourceful**)

* * *

### 17\. Перезапись updated_at во время сохранения

Знаете ли вы, что метод `->save()` может принимать параметры? Как результат мы можем “игнорировать” `updated_at`  функциональность, которая  по умолчанию должна была установить текущую метку времени.

Посмотрите на следующий пример:

    $product = Product::find($id);
    $product->updated_at = '2019-01-01 10:00:00';
    $product->save(['timestamps' => false]);
    

Здесь мы переписали `updated_at` нашем предустановленным значением.

* * *


### 18\. Что является результатом метода update()?


Вы когда-нибудь задумывались над тем, что этот код возвращает?

    $result = $products->whereNull('category_id')->update(['category_id' => 2]);
    

Я имею в виду, что обновление выполняется в базе данных, но что будет содержать этот `$result`?

Ответ: **затронутые строки**. Поэтому, если вам нужно проверить, сколько строк было затронуто, вам не нужно ничего вызывать - метод  `update()` вернет это число для вас.

* * *

### 19\. Преобразуем скобки в Eloquent запрос


Что делать если у вас есть AND и OR в вашем SQL запросе, как здесь:

    ... WHERE (gender = 'Male' and age >= 18) or (gender = 'Female' and age >= 65)
    

Как преобразовать этот запрос в Eloquent запрос? Это **неправильный** способ:

    $q->where('gender', 'Male');
    $q->orWhere('age', '>=', 18);
    $q->where('gender', 'Female');
    $q->orWhere('age', '>=', 65);
    

Порядок будет неправильным. Правильный способ немного сложнее, используя замыкания в качестве подзапросов:

    $q->where(function ($query) {
        $query->where('gender', 'Male')
            ->where('age', '>=', 18);
    })->orWhere(function($query) {
        $query->where('gender', 'Female')
            ->where('age', '>=', 65); 
    })
    

* * *


### 20\. orWhere с несколькими параметрами

Вы можете передать массив параметров в `orWhere()`.  
“Обычный” способ:

    $q->where('a', 1);
    $q->orWhere('b', 2);
    $q->orWhere('c', 3);
    

Вы можете сделать это так:

    $q->where('a', 1);
    $q->orWhere(['b' => 2, 'c' => 3]);
    

* * *


[Источник](https://laravel-news.com/eloquent-tips-tricks)