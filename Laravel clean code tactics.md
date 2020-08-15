# Laravel тактики написания кода

В этой теме я хочу показать набор тактик, которые помогут Вам писать более чистый код.
После того, как ты прочитаешь и проникнешься, у тебя появится нюх и ты сможешь определять запахи кода плохие и хорошие.

## 💡 Дьявол в деталях

Использование таких философий как Гексагональная архитектура или DDD не спасут вас.
Чистый код – это результат правильных решений на всех уровнях и мельчайших деталях.

## 👉 Ключ к массиву

Вместо того, чтобы писать огромную кучу условий `esle if`, используйте ключи массива как условия, а значения массива как результат того, что должно быть, если это ключ-условие верно.

**Плохо:**

```php
if ($order->product->option->type === 'pdf') {
    $type = 'book';
} else if ($order->product->option->type === 'epub') {
    $type = 'book';
} else if ($order->product->option->type === 'license') {
    $type = 'license';
} else if ($order->product->option->type === 'artwork') {
    $type = 'creative';
} else if ($order->product->option->type === 'song') {
    $type = 'creative';
} else if ($order->product->option->type === 'physical') {
    $type = 'physical';
}

if ($type === 'book') {
    $downloadable = true;
} else if ($type === 'license') {
    $downloadable = true;
} else if ($type === 'creative') {
    $downloadable = true;
} else if ($type === 'physical') {
    $downloadable = false;
}
```

**Хорошо:**

```php
$type = [
    'pdf' => 'book',
    'epub' => 'book',
    'license' => 'license',
    'artwork' => 'creative',
    'song' => 'creative',
    'physical' => 'physical',
][$order->product->option->type];

$downloadable = [
    'book' => true,
    'license' => true,
    'creative' => true,
    'physical' => false,
][$type];
```

## 👉 Отдай пораньше

Старайся избегать излишней вложенности, возвращая результат раньше.
Большая вложенность и условия `else` усложняют чтение кода.

**Плохо:**

```php
if ($notificationSent) {
   $notify = false;
} else {
    if ($isActive) {
        if ($total > 100) {
            $notify = true;
        } else {
            $notify = false;
        }
    } else {
        if ($canceled) {
            $notify = true;
        } else {
            $notify = false;
        }
    }
}

return $notify;
```

**Хорошо:**

```php
if ($notificationSent) {
    return false;
}

if ($isActive && $total > 100) {
    return true;
}

if (! $isActive && $canceled) {
    return true;
}

return false;
```

## 👉 Разделяй строки осмысленно

Не разбивай строки кода как попало, но и слишком длинные строки тоже плохо.

Открытие массива квадратной скобкой `[` и переносом строки читаются хорошо. Так же с функциями с большим кол-вом параметров.

По этому же принципу можно разбивать строки в цепочках вызовов и замыканиях.

**Плохо:**

```php
// Нет переносов
return $this->request->session()->get($this->config->get('analytics.campaign_session_key'));

// Переносы , но без смысла
return $this->request
    ->session()->get($this->config->get('analytics.campaign_session_key'));
```

**Хорошо:**

```php
return $this->request->session()->get(
    $this->config->get('analytics.campaign_session_key')
);

// Замыкание
new EventCollection($this->events->map(function (Event $event) {
    return new Entries\Event($event->code, $event->pivot->data);
}))

// Массив
$this->validate($request, [
    'code' => 'string|required',
    'name' => 'string|required',
]);
```

## 👉 Не плоди сущности

Не создавай переменных, когда можно сразу вернуть значение.

**Плохо:**

```php
public function create()
{
    $data = [
        'resource' => 'campaign',
        'generatedCode' => Str::random(8),
    ];

    return $this->inertia('Resource/Create', $data);
}
```

**Хорошо:**

```php
public function create()
{
    return $this->inertia('Resource/Create', [
        'resource' => 'campaign',
        'generatedCode' => Str::random(8),
    ]);
}
```

## 👉 Создавай переменные для улучшения читаемости

Иногда значения приходят из трудночитаемого вызова, и выгодно использовать переменные для улучшения читаемости, и это даже поможет избавиться от лишних комментариев в коде.

Запомни: все зависит от конкретной ситуации `И` если в итоге это легко читать

**Плохо:**

```php
Visit::create([
    'url' => $visit->url,
    'referer' => $visit->referer,
    'user_id' => $visit->userId,
    'ip' => $visit->ip,
    'timestamp' => $visit->timestamp,
])->conversion_goals()->attach($conversionData);
```

**Хорошо:**

```php
$visit = Visit::create([
    'url' => $visit->url,
    'referer' => $visit->referer,
    'user_id' => $visit->userId,
    'ip' => $visit->ip,
    'timestamp' => $visit->timestamp,
]);

$visit->conversion_goals()->attach($conversionData);
```

## 👉 Бизнес логика в Модели

Чем проще контроллеры, тем лучше. Они говорят что-то в этом духе "создать счет-фактуру на заказ". Они не должны содержать детали вашей базы данных.

Это как раз работа для Модели.

**Плохо:**

```php
// Создать счет-фактуру на заказ
DB::transaction(function () use ($order) {
    $invoice = $order->invoice()->create();

    $order->pushStatus(new AwaitingShipping);

    return $invoice;
});
```

**Хорошо:**

```php
$order->createInvoice();
```

## Отдельные классы для Действий

В дополнение к предыдущему примеру. Бывают случаи, когда созданием класса для каждого действия можно привести код в порядок.

Модель должна содержать свою бизнес логику, но не должна быть слишком большой.

**Плохо:**

```php
public function createInvoice(): Invoice
{
    if ($this->invoice()->exists()) {
        throw new OrderAlreadyHasAnInvoice('Order already has an invoice.');
    }

    return DB::transaction(function () use ($order) {
        $invoice = $order->invoice()->create();

        $order->pushStatus(new AwaitingShipping);

        return $invoice;
    });
}
```

**Хорошо:**

```php
// Модель Заказа
public function createInvoice(): Invoice
{
    if ($this->invoice()->exists()) {
        throw new OrderAlreadyHasAnInvoice('Order already has an invoice.');
    }

    return app(CreateInvoiceForOrder::class)($this);
}

// Класс Действия
class CreateInvoiceForOrder
{
    public function __invoke(Order $order): Invoice
    {
        return DB::transaction(function () use ($order) {
            $invoice = $order->invoice()->create();

            $order->pushStatus(new AwaitingShipping);

            return $invoice;
        });
    }
}
```

## 👉 Присмотрись к Форм Реквестам

Хорошее место, где можно спрятать сложную логику валидации запроса.

Будь осторожней с этими прятками. Когда логика валидации проста, нет ничего плохого в том, чтобы сделать это в контроллере.
Перенос простой логики в Форм Реквест может сделать код менее очевидным.

```php
/**
 * Правила валидации для запроса.
 *
 * @return array
 */
public function rules()
{
    return [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
    ];
}
```

## 👉 События пригодятся

Также неплохо отгрузить немного логики из контроллеров в события. Например, создание моделей.

Плюс в том, что создание моделей будет работать одинаково везде (Контроллеры, Задачи(Jobs), и т.п), и контроллер снова не знает, как устроена наша база данных.

**Плохо:**

```php
// Работает только здесь и касается детали 
// которые должны быть в компетенции модели.
if (! isset($data['name'])) {
    $data['name'] = $data['code'];
}

$conversion = Conversion::create($data);
```

**Хорошо:**

```php
$conversion = Conversion::create($data);

// Model
class ConversionGoal extends Model
{
    public static function booted()
    {
        static::creating(function (self $model) {
            $model->name ??= $model->code;
        });
    }
}
```

## 👉 Выжимай лишнее из метода

Если метод слишком большой или сложный, и сходу его не понять, раздели его на несколько методов.

```php
public function handle(Request $request, Closure $next)
{
    // We extracted 3 long methods into separate methods.
    $this->trackVisitor();
    $this->trackCampaign();
    $this->trackTrafficSource($request);

    $response = $next($request);

    $this->analytics->log($request);

    return $response;
}
```

## 👉 Функции помощники

Если используешь один и тот же кусок кода много раз, значит пора выделить его в отдельную функцию-помощник. Это сделает код чище.

```php
// app/helpers.php, autoloaded in composer.json

function money(int $amount, string $currency = null): Money
{
    return new Money($amount, $currency ?? config('shop.base_currency'));
}

function html2text($html = ''): string
{
    return str_replace(' ', ' ', strip_tags($html));
}
```

## 👉 Остерегайся **Классов** помощников

Иногда люди добавляют хелперы в Классы.

Остерегайся, возможен бардак. Это класс с одними только статическими методами в виде функций помощников. Лучше добавлять эти методы в классы со схожей логикой или просто оставлять их глобальными функциями.

**Плохо:**

```php
class Helper
{
    public function convertCurrency(Money $money, string $currency): self
    {
                // Конвертируем из текущей валюты в базовую, а затем в новую валюту
        $newCurrencyConfig = config("shop.currencies.$currency");
        $mathDecimalDifference = $newCurrencyConfig['math_decimals'] - config("shop.currencies." . config('shop.base_currency') . ".math_decimals");

        return new static(
            (int) round($money->baseValue() * $newCurrencyConfig['value'] * 10**$mathDecimalDifference, 0),
            $currency
        );
    }
}

// Использование:
use App\Helper;

Helper::convertCurrency($total, 'EUR');
```

**Хорошо:**

```php
class Money
{
    // здесь другая денежная/валютная логика

   public function convertTo(string $currency): self
    {
        // Конвертируем из текущей валюты в базовую, а затем в новую валюту
        $newCurrencyConfig = config("shop.currencies.$currency");
        $mathDecimalDifference = $newCurrencyConfig['math_decimals'] - config("shop.currencies." . config('shop.base_currency') . ".math_decimals");

        return new static(
            (int) round($this->baseValue() * $newCurrencyConfig['value'] * 10**$mathDecimalDifference, 0),
            $currency
        );
    }
}

// Использование:
$EURtotal = $total->convertTo('EUR');
```

## 💡 Выдели пару дней для изучения правильного ООП.

Знать разницу между статическими методами и методами объекта `&` переменные и области видимости (private/protected/public). Изучить как работают магические методы в Laravel.

В самом начале это не так важно, но позже это когда кода станет больше это очень важно.

## 💡 Не пиши процедурный код в классах

Это связывает предыдущий и следующие советы. ООП существует для того, чтобы сделать твой код более читаемым, используй это. Не пиши 400 строк процедурного кода прямо в лоб в контроллере.

## 💡 Почитай о таких вещах как Принцип Единой Ответственности & следуй им тогда когда это *разумно*

Избегай классов, которые работают со множеством несвязанных вещей.

Ради бога, не создавайте кучу классов на каждый чих. Вы стараетесь сделать код чище, а не угодить богу Единой Ответственности.

## 💡 Почему у функции много параметров:

1. У функции много ответственностей. Разделяй.
2. С ответственностями все в порядке, но Вам нужно отрефакторить объявление функции.

Ниже пара способов как поступить.

## 👉 Использовать Объекты Передачи Данных(ДТОхи)

Зачем пробрасывать большое количество аргументов да еще и в определенном порядке, 
присмотритесь к тому, чтобы создать объект для хранения этих аргументов.

Бонусные очки, если вы сможете найти поведение, которое можно перенести в этот объект.

**Плохо:**

```php
public function log($url, $route_name, $route_data, $campaign_code, $traffic_source, $referer, $user_id, $visitor_id, $ip, $timestamp)
{
    // ...
}
```

**Хорошо:**

```php
public function log(Visit $visit)
{
    // ...
}

class Visit
{
    public string $url;
    public ?string $routeName;
    public array $routeData;

    public ?string $campaign;
    public array $trafficSource = [];

    public ?string $referer;
    public ?string $userId;
    public string $visitorId;
    public ?string $ip;

    public Carbon $timestamp;

    // ...
}
```

## 👉 Создавайте текучие(плавные) объекты

Вы можете создавать объекты с текущим API. 
Постепенно добавляя аргументы отдельными вызовами, а в конструкторе объекта используйте только необходимые аргументы.

Каждый метод должен возвратить $this, так что вы сможете остановиться на любом вызове.

```php
Visit::make($url, $routeName, $routeData)
    ->withCampaign($campaign)
    ->withTrafficSource($trafficSource)
    ->withReferer($referer)
    // ... и т.д
```

## 👉 Создавайте свои Коллекции

Создание своих собственных методов Коллекций – это лучший способ получить более выразительный синтаксис.
Как например с этой Коллекцией Заказов.

**Плохо:**

```php
$total = $order->products->sum(function (OrderProduct $product) {
    return $product->price * $product->quantity * (1 + $product->vat_rate);
});
```

**Хорошо:**

```php
$order->products->total();

class OrderProductCollection extends Collection
{
    public function total()
    {
        $this->->sum(function (OrderProduct $product) {
            return $product->price * $product->quantity * (1 + $product->vat_rate);
        });
    }
}
```

## 👉 Пиши не сокращай

Не думай, что длинные имена переменных/методов – это плохо. Это не так. Это выразительно.

Лучше вызвать длинный метод, чем постоянно заглядывать в док-блок, чтобы понять, что этот метод делает.

Тоже и с переменными. Не пиши переменные из трех букв.

**Плохо:**

```php
$ord = Order::create($data);

// ...

$ord->notify();
```

**Хорошо:**

```php
$order = Order::create($data);

// ...

$order->sendCreatedNotification();
```

## 👉 Постарайся использовать только CRUD методы

Старайся использовать 7 стандартных CRUD методов в контроллере или и того меньше.
Не создавай контроллер на 20 методов. 
Лучше создать отдельный контроллер `PodcastSubscriptionController::store()`
чем добавлять метод `PodcastController::subscribe()` в уже существующий контроллер.

["Cruddy by Design" - Adam Wathan - Laracon US 2017](https://www.youtube.com/watch?v=MF0jFKvS4SI)

## 👉 Именуй методы правильно

Лучше думай так "что можно сделать с этим объектом", чем "что этот объект может сделать".

Бывают и исключения, например с классами действий, но от это не делает это правило плохим.

["Resisting Complexity" - Adam Wathan - Laracon US 2018](https://www.youtube.com/watch?v=dfgtKb-VpRk)

**Плохо:**

```php
$gardener->water($plant);

$orderManager->lock($order);
```

**Хорошо:**

```php
$plant->water();

$order->lock();
```

## 👉 Одноразовые Трейты

Добавлять методы в классы, где они нужны – это лучше, чем создавать по классу на каждое действие, но из-за этого класс может разрастись.

Возможно стоит использовать Трейт. Это не значит, что трейт *обязательно* должен быть переиспользован, нет ничего плохого в том, чтобы использовать трейт одноразово.
**Плохо:**

```php
class Order extends Model
{
    // ...

    public static function bootHasStatuses() { ... }
    public static $statusMap = [ ... ];
    public static $paymentStatusMap = [ ... ];
    public static function getStatusId($status) { ... }
    public static function getPaymentStatusId($status): string { ... }
    public function statuses() { ... }
    public function payment_statuses() { ... }
    public function getStatusAttribute(): OrderStatusModel { ... }
    public function getPaymentStatusAttribute(): OrderPaymentStatus { ... }
    public function pushStatus($status, string $message = null, bool $notification = null) { ... }
    public function pushPaymentStatus($status, string $note = null) { ... }
    public function status(): OrderStatus { ... }
    public function paymentStatus(): PaymentStatus { ... }
}
```

**Хорошо:**

```php
class Order extends Model
{
    use HasStatuses;

    // ...
}

trait HasStatuses
{
    public static function bootHasStatuses() { ... }
    public static $statusMap = [ ... ];
    public static $paymentStatusMap = [ ... ];
    public static function getStatusId($status) { ... }
    public static function getPaymentStatusId($status): string { ... }
    public function statuses() { ... }
    public function payment_statuses() { ... }
    public function getStatusAttribute(): OrderStatusModel { ... }
    public function getPaymentStatusAttribute(): OrderPaymentStatus { ... }
    public function pushStatus($status, string $message = null, bool $notification = null) { ... }
    public function pushPaymentStatus($status, string $note = null) { ... }
    public function status(): OrderStatus { ... }
    public function paymentStatus(): PaymentStatus { ... }
}
```

## 👉 Одноразовый Блэйд @include

Та же тема, что и с одноразовыми трейтами.

Эта тактика хороша когда у тебя слишком длинный шаблон и ты хочешь сделать его 
более читаемым.

Нет ничего плохого в том чтобы инклюдить хедер или футер в шаблоны, или к примеру сложную форму на страницу.

## 👉 Импорт Пространств имен вместо Псевдонимов

Иногда имена классов могут совпадать. Тут можно вместо псевдонимов (алиасов) использовать импорт пространств имен (неймспейсов).

**Плохо:**

```php
use App\Types\Entries\Visit as VisitEntry;
use App\Storage\Database\Models\Visit as VisitModel;

class DatabaseStorage
{
    public function log(VisitEntry $visit)
    {
        $visitModel = VisitModel::create([
            ...
        ]);
    }
}
```

**Хорошо:**

```php
use App\Types\Entries;
use App\Storage\Database\Models;

class DatabaseStorage
{
    public function log(Entries\Visit $visit)
    {
        $visitModel = Models\Visit::create([
            ...
        ]);
    }
}
```

## 👉 Создать Скоупы для сложных запросов Where()s

Зачем писать сложные запросы where(), создай Скоуп для запроса с говорящим именем.
Это может сделать код чище, а контроллер, к примеру, может меньше знать о базе данных.

**Плохо:**

```php
Order::whereHas('status', function ($status) {
    $status->where('canceled', true);
})->get();
```

**Хорошо:**

```php
Order::whereCanceled()->get();

class Order extends Model
{
    public function scopeWhereCanceled(Builder $query)
    {
        $query->whereHas('status', function ($status) {
            $status->where('canceled', true);
        })->get();
    }
}
```

## 👉 Не используй Методы Модели для извлечения данных

Если нужно достать данные из модели, создай акцессор. Оставь методы для того, чтобы *менять* модель определенным образом.

**Плохо:**

```php
$user->gravatarUrl();

class User extends Authenticable
{
    // ...

    public function gravatarUrl()
    {
        return "https://www.gravatar.com/avatar/" . md5(strtolower(trim($this->email)));
    }
}
```

**Хорошо:**

```php
$user->gravatar_url;

class User extends Authenticable
{
    // ...

    public function getGravatarUrlAttribute()
    {
        return "https://www.gravatar.com/avatar/" . md5(strtolower(trim($this->email)));
    }
}
```

## 👉 Используй собственные конфиг файлы

Ты можешь сохранять такие настройки, как "товаров на странице" в файлах конфигурации.
Не добавляй их в `app` файл конфигурации. Создай отдельный файл настроек. В моем приложении для онлайн-магазина я использую config/shop.php.

```php
return [
    'vat_rates' => [
        0.21,
        0.15,
        0.10,
        0.0,
    ],
    'fee_vat_rate' => 0.21,

    'image_sizes' => [
        'base' => 500, // detail
        't1' => 250, // index
        't2' => 50, // search
    ],
];
```

## 👉 Не используй пространство имен контроллера

Вместо того, чтобы писать действия контроллера как `PostController@index`, используйте массив `[PostController::class, 'index']`. Ты сможешь перейти в класс контроллера по клику на `PostController`.

**Плохо:**

```php
Route::get('/posts', 'PostController@index');
```

**Хорошо:**

```php
Route::get('/posts', [PostController::class, 'index']);
```

## 👉 Присмотрись к Контролерам с одним действием

Если у Вас есть сложное действие в маршруте, рассмотри возможность перенести его в отдельный контроллер.

Для `OrderController::create` вы создаете `CreateOrderController`.

Другое решение – это перенести эту логику в экшен класс. Делай то, что лучше в твоем случае.

```php
// Используем синтаксис классов из совета выше.
Route::post('/orders/', CreateOrderController::class);

class CreateOrderController
{
    public function __invoke(Request $request)
    {
        // ...
    }
}
```

## 👉 Подружись со своей IDE

Установи расширения, пиши аннотации, используй подсказки типов. Твоя IDE поможет тебе в том чтобы твой код работал правильно, 
Install extensions, write annotations, use typehints. Your IDE will help you with getting your code working correctly, что позволяет Вам вложить больше энергии в написание кода.

```php
$products = Product::with('options')->cursor();

foreach ($products as $product) {
    /** @var Product $product */

    if ($product->options->isEmpty()) {

///////////////////////////

foreach (Order::whereDoesntHave('invoice')->whereIn('id', $orders->pluck('id'))->get() as $order) {
    /** @var Order $order */
    $order->createInvoice();
} 

///////////////////////////

->help('Max 2 MB')
->store(function (NovaRequest $request, ProductModel $product) {
    /** @var UploadedFile $image */
    $image = $request->image;
```

## 👉 Используйте короткие операторы

у PHP есть много чудесных операторов которые могут заменить уродливые проверки if-ами. Запомни их.
PHP has many great operators that can replace ugly if checks. Memorize them.

```php
// тест на истинность

// ❌                     // ✅
if (! $foo) {             $foo = $foo ?: 'bar';
    $foo = 'bar';
}

// тест на нул

// ❌                     // ✅
if (is_null($foo)) {      $foo = $foo ?? 'bar';
    $foo = 'bar';
}                         // ✅ PHP 7.4
                          $foo ??= 'bar';

// установлено ли значение

// ❌                     // ✅
if (! isset($foo)) {      $foo = $foo ?? 'bar';
    $foo = 'bar';
}                         // ✅ PHP 7.4
                          $foo ??= 'bar';
```

## 👉 Подумай на тем чтобы отделять операторы пробелами

В примере выше вы могли обратить внимание на то, что я использую пробелы между ! и значением, которое отрицаю. Мне это нравится, поскольку это проясняет, что значение отрицается. Я делаю то же и с точками.

Подумай, может тебе тоже это понравится. Это сделает (имхо) твой код чище.

## 👉 Рассмотри использование Хелперов вместо Фасадов

Это больше личное предпочтение, но вызов глобальной функции вместо импорта класса и статического вызова метода по мне лучше.

Бонусные очки за синтаксис session('key').

```php
// ❌
Cache::get('foo');

// ✅
cache()->get('foo');

// 😍
cache('foo');
```

## 👉 Создавай собственные Блэйд-директивы для бизнес логики

Вы можете сделать шаблон Блейд более выразительным если создадите собственные директивы. Для примера, вместо того чтобы проверять роль админа, можно использовать директиву @admin.

**Плохо:**

```php
@if(auth()->user()->hasRole('admin'))
    // ...
@else
    // ...
@endif
```

**Хорошо:**

```php
@admin
    // ...
@else
    // ...
@endadmin
```

## 👉 Избегай запросов в Блейд шаблонах

Иногда вы можете захотеть использовать запросы к БД в шаблоне Блейд. 
В некоторых случаях это нормально.

Но если Вьюха приходит из Контроллера, то просто пробросьте необходимые данные в нее сразу в контроллере.

**Плохо:**

```php
@foreach(Product::where('enabled', false)->get() as $product)
    // ...
@endforeach
```

##### **Хорошо:**

```php
// Controller
return view('foo', [
    'disabledProducts' => Product::where('enabled', false)->get(),
]);

// View
@foreach($disabledProducts as $product)
    // ...
@endforeach
```

## 👉 Используйте строгое сравнение

ВСЕГДА используйте строгое сравнение (=== и !==). Если нужно, меняйте тип данных перед сравнением. Это лучше, чем небезопасный результат от ==

Также рассмотрите включение строгой типизации в вашем коде. Это должно предотвратить
пробрасывание данных с неправильным типом в переменные.

```php
// ❌
$foo == 'bar';

// ✅
$foo === 'bar';

// 💪
declare(strict_types=1);
```

## 👉 Используй док-блоки только когда они что-то проясняют

Многие люди не согласятся с этим, они делают иначе. Но в этом нет смысла.

Нет смысла в использовании док-блоков когда в них нет никакой дополнительной информации. Если подсказки типов достаточно, не используй док-блоки.

Они создают лишь шум.

```php
// 🤮 Нет типов вообще
function add_5($foo)
{
    return $foo + 5;
}

// ✅ Все без док-блоков
function add_5(int $foo)
{
    return $foo + 5;
}

// ❌ Аннотация @param добавляет 0% выгоды и 100% шума.

/**
 * сложить 5 и число.
 *
 * @param int $foo
 * @return int
 */
function add_5(int $foo): int
{
    return $foo + 5;
}

// 😎 В подсказках типов сказано много, а в аннотации еще больше.

/**
 * Turn words into a sentence.
 *
 * @param string[] $words
 * @return string
 */
function sentenceFromWords(array $words): string
{
    return implode(' ', $words) . '.';
}

// 🚀 Мое личное предпочтение. 
// Использовать только аннотации которые приносят выгоду.
// Не используй description - param - return просто потому что это стандарт.

/** @param string[] $words */
function sentenceFromWords(array $words): string
{
    return implode(' ', $words) . '.';
}
```

## 👉 Один источник истины для всех привил валидации

Имей один источник истины для всех привил валидации.

Если ты валидируешь атрибуты какого-то ресурса в нескольких местах, ты определенно хочешь централизовать это правило валидации, чтобы, если поменял его в одном месте, не забыть про остальные места.

Модели могут быть хорошим местом для это.

[https://twitter.com/LiamHammett/status/1260252814158282752](https://twitter.com/LiamHammett/status/1260252814158282752)

## 👉 Используй коллекции когда они делают твой код чище

Не нужно из всех массивов делать коллекции просто потому что в Ларавел они есть,

но ДЕЛАЙ из массивов коллекции когда использование синтаксиса коллекций делает код чище.

```php
$collection = collect([
    ['name' => 'Regena', 'age' => null],
    ['name' => 'Linda', 'age' => 14],
    ['name' => 'Diego', 'age' => 23],
    ['name' => 'Linda', 'age' => 84],
]);

$collection->firstWhere('age', '>=', 18);
```

## 👉 Пиши в функциональном стиле когда это выгодно

Функциональный код может сделать вещи чище и упростить их понимание.

Рефактори циклы в функциональные вызов, но не пиши излишне сложные редьюсы

только чтобы избежать цикла. Вот пара примеров.

**Хорошее ФП:**

```php
return $this->items()->reduce(function (Money $sum, OrderItem $item) {
    return $sum->addMoney($item->subtotal());
}, money(0, $this->currency));
```

**Плохое ФП:**

```php
// Пример из того что писал я 🙃

return array_unique(array_reduce($keywords, function ($result, $keyword) {
    return array_merge($result, array_reduce($this->variantGenerators, function ($result2, $generator) use ($keyword) {
        return array_merge($result2, array_map(function ($variant) {
            return strtolower($variant);
        }, $generator::getVariants($keyword)));
    }, []));
}, []));
```

## 💡 Комментарии обычно указывают на слабый дизайн кода

Перед тем как написать коммент, спроси себя: можешь ли ты переименовать или создать переменную, чтобы прояснить код. Если это не возможно, тогда пиши комментарий, но только так, чтобы ты и твои коллеги смогли понять его и через 6 месяцев.

## 💡 Контекст имеет значение

Выше я говорил что перенесение бизнес логики в отдельные действия/сервисы это хорошо. Но контекст имеет значение.

Здесь код советы по написанию кода из репозитория "Larave best practices".

Нет абсолютно ни какого смысла выносить три строчки с проверками в отдельный класс. Это просто оверинжиниринг.

https://twitter.com/samuelstancl/status/1272826465378357251

## 💡 Используй только то что тебе помогает, остальное игнорируй

Этот совет дополняет предыдущий твит. *Ваша цель писать более читаемый код.* *Ваша цель НЕ в том, чтобы делать то, что кто-то сказал в интернете*.

Это советы просто тактики которые помогают сделать код чище. Помни о своей конечной цели, задай себе вопрос: "Это решение лучше ?".

[Автор Оригинала: Samuel Štancl](https://threadreaderapp.com/thread/1272822437181378561.html)

[Автор перевода: Vuwave](https://github.io/vuwav)
