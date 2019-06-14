![Beste Laravel gebruiken](/images/logo-nl.png)

[Origineel in Engels](https://github.com/alexeymezenin/laravel-best-practices) (by [alexeymezenin](https://github.com/alexeymezenin))

Hier vind je de beste programmeer praktijken voor Laravel.

## **Inhoudsopgave**

[Enkel verantwoordelijkheidsprincipe](#enkel-verantwoordelijkheidsprincipe)

[Uitgebreidere modellen, lichtere controllers](#uitgebreidere-modellen-lichtere-controllers)

[Validatie](#validatie)

[Bedrijfslogica past het beste in een `Service` klasse](#bedrijfslogica-past-het-beste-in-een-service-klasse)

[Herhaal jezelf niet - Don't repeat yourself (DRY)](#herhaal-jezelf-niet-dont-repeat-yourself-dry)

[Prefereer Eloquent boven de Query Builder en prefereer collecties boven arrays](#prefereer-eloquent-boven-de-query-builder-en-prefereer-collecties-boven-arrays)

[In een keer veel data toevoegen](#in-een-keer-veel-data-toevoegen)

[Voer geen queries uit in Blade Templates en gebruik `eager loading` om N+1 te voorkomen](#voer-geen-queries-uit-in-blade-templates-en-gebruik-eager-loading-om-n1-te-voorkomen)

[Geef commentaar bij je code, maar prefereer een beschrijvende functie/methode naam en variabelen als commentaar](#geef-commentaar-bij-je-code-maar-prefereer-een-beschrijvende-functiemethode-naam-en-variabelen-als-commentaar)

[Gebruik niet direct JavaScript of CSS in je Blade Templates en gebruik geen HTML in je PHP klassen](#gebruik-niet-direct-javascript-of-css-in-je-blade-templates-en-gebruik-geen-html-in-je-php-klassen)

[Gebruik de `config` en `language` bestanden en constantes in plaats van tekst in je code](#gebruik-de-config-en-language-bestanden-en-constantes-in-plaats-van-tekst-in-je-code)

[Gebruik de standaard Laravel gereedschappen die geaccepteerd zijn door de gemeenschap](#gebruik-de-standaard-laravel-gereedschappen-die-geaccepteerd-zijn-door-de-gemeenschap)

[Gebruik Laravel naam conventies](#gebruik-laravel-naam-conventies)

[Gebruik kortere en meer leesbare syntax waar mogelijk](#gebruik-kortere-en-meer-leesbare-syntax-waar-mogelijk)

[Gebruik `IoC` containers of facades in plaats van een nieuwe klasse](#gebruik-ioc-containers-of-facades-in-plaats-van-een-nieuwe-klasse)

[Haal niet direct data op uit het `.env` bestand](#haal-niet-direct-data-op-uit-het-env-bestand)

[Sla datums op in een standaard formaat en gebruik `accessors` en `mutators` om deze data aan te passen](#sla-datums-op-in-een-standaard-formaat-en-gebruik-accessors-en-mutators-om-deze-data-aan-te-passen)

[Andere goede gebruiken](#andere-goede-gebruiken)

### **Enkel verantwoordelijkheidsprincipe**

Een klasse en een methode zouden maar 1 verantwoordelijkheid moeten hebben.

Afgeraden:

```php
public function getFullNameAttribute()
{
    if (auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified()) {
        return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
    } else {
        return $this->first_name[0] . '. ' . $this->last_name;
    }
}
```

Beter:

```php
public function getFullNameAttribute()
{
    return $this->isVerifiedClient() ? $this->getFullNameLong() : $this->getFullNameShort();
}

public function isVerifiedClient()
{
    return auth()->user() && auth()->user()->hasRole('client') && auth()->user()->isVerified();
}

public function getFullNameLong()
{
    return 'Mr. ' . $this->first_name . ' ' . $this->middle_name . ' ' . $this->last_name;
}

public function getFullNameShort()
{
    return $this->first_name[0] . '. ' . $this->last_name;
}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Uitgebreidere modellen, lichtere controllers**

Stop all je Database gerelateerde in `Eloquent` modellen of als je de Query Builder gebruikt in `Repository` klassen.

Afgeraden:

```php
public function index()
{
    $clients = Client::verified()
        ->with(['orders' => function ($q) {
            $q->where('created_at', '>', Carbon::today()->subWeek());
        }])
        ->get();

    return view('index', ['clients' => $clients]);
}
```

Beter:

```php
public function index()
{
    return view('index', ['clients' => $this->client->getWithNewOrders()]);
}

class Client extends Model
{
    public function getWithNewOrders()
    {
        return $this->verified()
            ->with(['orders' => function ($q) {
                $q->where('created_at', '>', Carbon::today()->subWeek());
            }])
            ->get();
    }
}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Validatie**

Verplaats validatie uit controllers naar daarvoor bestemde `Request` klassen.

Afgeraden:

```php
public function store(Request $request)
{
    $request->validate([
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ]);

    ....
}
```

Beter:

```php
public function store(PostRequest $request)
{    
    ....
}

class PostRequest extends Request
{
    public function rules()
    {
        return [
            'title' => 'required|unique:posts|max:255',
            'body' => 'required',
            'publish_at' => 'nullable|date',
        ];
    }
}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Bedrijfslogica past het beste in een `Service` klasse**

Zoals eerder genoemd zou een controller maar 1 verantwoordelijkheid moeten hebben. Verplaats daarom je bedrijfslogica naar een `Service` klasse.

Afgeraden:

```php
public function store(Request $request)
{
    if ($request->hasFile('image')) {
        $request->file('image')->move(public_path('images') . 'temp');
    }
    
    ....
}
```

Beter:

```php
public function store(Request $request)
{
    $this->articleService->handleUploadedImage($request->file('image'));

    ....
}

class ArticleService
{
    public function handleUploadedImage($image)
    {
        if (!is_null($image)) {
            $image->move(public_path('images') . 'temp');
        }
    }
}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Herhaal jezelf niet - Don't repeat yourself (DRY)**

Herbruik code wanneer mogelijk. Het 'Enkel verantwoordelijksheidprincipe' helpt je al bij het vermijden van gedupliceerde code. Herbruik ook Blade Templates, en gebruik Eloquent scopes.

Afgeraden:

```php
public function getActive()
{
    return $this->where('verified', 1)->whereNotNull('deleted_at')->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->where('verified', 1)->whereNotNull('deleted_at');
        })->get();
}
```

Beter:

```php
public function scopeActive($q)
{
    return $q->where('verified', 1)->whereNotNull('deleted_at');
}

public function getActive()
{
    return $this->active()->get();
}

public function getArticles()
{
    return $this->whereHas('user', function ($q) {
            $q->active();
        })->get();
}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Prefereer Eloquent boven de Query Builder en prefereer collecties boven arrays**

Eloquent geeft je de mogelijkheid leesbaar- en bij te houden code te schrijven. Eloquent heeft hiervoor ook erg gemakkelijke standaard gereedschappen zoals o.a. 'Soft-deletes', 'Events', 'Scopes' en nog veel meer.

Afgeraden:

```sql
SELECT *
FROM `articles`
WHERE EXISTS (SELECT *
              FROM `users`
              WHERE `articles`.`user_id` = `users`.`id`
              AND EXISTS (SELECT *
                          FROM `profiles`
                          WHERE `profiles`.`user_id` = `users`.`id`) 
              AND `users`.`deleted_at` IS NULL)
AND `verified` = '1'
AND `active` = '1'
ORDER BY `created_at` DESC
```

Beter:

```php
Article::has('user.profile')->verified()->latest()->get();
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **In een keer veel data toevoegen**

Afgeraden:

```php
$article = new Article;
$article->title = $request->title;
$article->content = $request->content;
$article->verified = $request->verified;
$article->category_id = $category->id;
$article->save();
```

Beter:

```php
$category->article()->create($request->validated());
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Voer geen queries uit in Blade Templates en gebruik eager loading om N+1 te voorkomen**

Afgeraden (voor 100 gebruikers, worden er 101 DB queries uitegevoerd):

```php
@foreach (User::all() as $user)
    {{ $user->profile->name }}
@endforeach
```

Beter (voor 100 gebruikers, 2 DB queries zullen worden uitgevoerd):

```php
$users = User::with('profile')->get();

...

@foreach ($users as $user)
    {{ $user->profile->name }}
@endforeach
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Geef commentaar bij je code, maar prefereer een beschrijvende functie/methode naam en variabelen als commentaar**

Afgeraden:

```php
if (count((array) $builder->getQuery()->joins) > 0)
```

Better:

```php
// Check of er joins voorkomen in de query
if (count((array) $builder->getQuery()->joins) > 0)
```

Beter:

```php
if ($this->hasJoins())
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik niet direct JavaScript of CSS in je Blade Templates en gebruik geen HTML in je PHP klassen**

Afgeraden:

```php
let article = `{{ json_encode($article) }}`;
```

Better:

```php
<input id="article" type="hidden" value="@json($article)">

Or

<button class="js-fav-article" data-article="@json($article)">{{ $article->name }}<button>
```

In een Javascript bestand:

```javascript
let article = $('#article').val();
```

De beste manier is om een gespecialiseerd PHP naar JavaScript package te gebruiken om data door te voeren.

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik de config en language bestanden en constantes in plaats van tekst in je code**


Afgeraden:

```php
public function isNormal()
{
    return $article->type === 'normal';
}

return back()->with('message', 'Your article has been added!');
```

Beter:

```php
public function isNormal()
{
    return $article->type === Article::TYPE_NORMAL;
}

return back()->with('message', __('app.article_added'));
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik de standaard Laravel gereedschappen die geaccepteerd zijn door de gemeenschap**

Geef voorkeur aan ingebouwde Laravel functionaliteit of packages door de Laravel gemeenschap in plaats van packages van 3de partijen. Hierdoor kan je applicatie beter worden bijgehouden door andere programmeurs en kan je tijd besparen doordat er vaak online een antwoord gevonden kan worden als je problemen ondervindt. 

Doelen | Standaard middelen/packages | Middelen/Packages van 3de partijen
------------ | ------------- | -------------
Authorisatie | Policies | Entrust, Sentinel en andere packages
Compileren van je assets | Laravel Mix | Grunt, Gulp en andere packages
Ontwikkel omgeving | Homestead | Docker
Uitrollen/Deployment | Laravel Forge | Deployer en andere oplossingen
Unit testen | PHPUnit, Mockery | Phpspec
Browser testen | Laravel Dusk | Codeception
Database | Eloquent | SQL, Doctrine
Sjablonen/Templates | Blade | Twig
Werken met data | Laravel collections | Arrays
Formulier validatie | Request classes | Packages van 3de partijen, validatie in controller
Authenticatie | Ingebouwd | Packages van 3de partijen of je eigen oplossing
API authenticatie | Laravel Passport | 3de partij packages JWT en/of OAuth 
Bouwen van een API | Ingebouwd | Dingo API en andere soortgelijke packages
Werken met een database structuur | Migrations | Direct met database structuur werken
Localisatie/I18N | Ingebouwd | Packages van 3de partijen
'Realtime' gebruikersomgeving | Laravel Echo, Pusher | Packages van 3de partijen of direct met WebSockets werken
Test data genereren | Seeder classes, Model Factories, Faker | Zelf maken van test data
Taken inplannen| Laravel Task Scheduler | Scripten en packages van 3de partijen
DB | MySQL, PostgreSQL, SQLite, SQL Server | MongoDB

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik Laravel naam conventies**

 Gebruik [PSR standaarden (EN)](http://www.php-fig.org/psr/psr-2/).
 
 Gebruik ook naam conventies geaccepteerd door de Laravel gemeenschap:

Wat | Hoe | Goed | Afgeraden
------------ | ------------- | ------------- | -------------
Controller | enkelvoud | ArticleController | ~~ArticlesController~~
Route | meervoud | articles/1 | ~~article/1~~
Benoemde route | 'snake_case' met punt notatie | users.show_active | ~~users.show-active, show-active-users~~
Model | enkelvoud | User | ~~Users~~
hasOne of belongsTo relatie | enkelvoud | articleComment | ~~articleComments, article_comment~~
Alle andere relaties | meervoud | articleComments | ~~articleComment, article_comments~~
Tabel | meervoud | article_comments | ~~article_comment, articleComments~~
Pivot table | enkelvoud model namen in alfabetische volgorde | article_user | ~~user_article, articles_users~~
Table kolom | 'snake_case' zonder model naam | meta_title | ~~MetaTitle; article_meta_title~~
Model eigenschap | snake_case | $model->created_at | ~~$model->createdAt~~
Sleutel buiten huidige tabel | enkelvoud model naam met een _id achtervoegsel | article_id | ~~ArticleId, id_article, articles_id~~
Primaire sleutel | - | id | ~~custom_id~~
Migratie | - | 2017_01_01_000000_create_articles_table | ~~2017_01_01_000000_articles~~
Methode | camelCase | getAll | ~~get_all~~
Methode in een resource controller | [tabel](https://laravel.com/docs/master/controllers#resource-controllers) | store | ~~saveArticle~~
Methode in test class | camelCase | testGuestCannotSeeArticle | ~~test_guest_cannot_see_article~~
Variabele | camelCase | $articlesWithAuthor | ~~$articles_with_author~~
Collectie | beschrijvend, meervoud | $activeUsers = User::active()->get() | ~~$active, $data~~
Object | beschrijvend, enkelvoud | $activeUser = User::active()->first() | ~~$users, $obj~~
Configuratie en taal bestanden index | snake_case | articles_enabled | ~~ArticlesEnabled; articles-enabled~~
View | snake_case | show_filtered.blade.php | ~~showFiltered.blade.php, show-filtered.blade.php~~
Config | snake_case | google_calendar.php | ~~googleCalendar.php, google-calendar.php~~
Contract (interface) | 
bijvoeglijk naamwoord of werkwoord | Authenticatable | ~~AuthenticationInterface, IAuthentication~~
Trait | bijvoeglijk naamwoord | Notifiable | ~~NotificationTrait~~

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik kortere en meer leesbare syntax waar mogelijk**

Afgeraden:

```php
$request->session()->get('cart');
$request->input('name');
```

Beter:

```php
session('cart');
$request->name;
```

Meer voorbeelden:

Gebruikelijke syntax | Korter- en meer leesbare syntax
------------ | -------------
`Session::get('cart')` | `session('cart')`
`$request->session()->get('cart')` | `session('cart')`
`Session::put('cart', $data)` | `session(['cart' => $data])`
`$request->input('name'), Request::get('name')` | `$request->name, request('name')`
`return Redirect::back()` | `return back()`
`is_null($object->relation) ? null : $object->relation->id` | `optional($object->relation)->id`
`return view('index')->with('title', $title)->with('client', $client)` | `return view('index', compact('title', 'client'))`
`$request->has('value') ? $request->value : 'default';` | `$request->get('value', 'default')`
`Carbon::now(), Carbon::today()` | `now(), today()`
`App::make('Class')` | `app('Class')`
`->where('column', '=', 1)` | `->where('column', 1)`
`->orderBy('created_at', 'desc')` | `->latest()`
`->orderBy('age', 'desc')` | `->latest('age')`
`->orderBy('created_at', 'asc')` | `->oldest()`
`->select('id', 'name')->get()` | `->get(['id', 'name'])`
`->first()->name` | `->value('name')`

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Gebruik IoC containers of facades in plaats van een nieuwe klasse**

Nieuwe klasse syntax creeert een nauwe koppeling tussen klassen en compliceert daarom testen. Gebruik daarom een IoC container of facades.

Afgeraden:

```php
$user = new User;
$user->create($request->validated());
```

Beter:

```php
public function __construct(User $user)
{
    $this->user = $user;
}

....

$this->user->create($request->validated());
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Haal niet direct data op uit het `.env` bestand**

Voer de de eerder door naar de configuratie bestanden (`config/*.php`) en gebruik dan de `config()` helper om data te gebruiken in je programma.

Afgeraden:

```php
$apiKey = env('API_KEY');
```

Beter:

```php
// config/api.php
'key' => env('API_KEY'),

// De data gebruiken 
$apiKey = config('api.key');
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Sla datums op in een standaard formaat en gebruik accessors en mutators om deze data aan te passen**

Afgeraden:

```php
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->toDateString() }}
{{ Carbon::createFromFormat('Y-d-m H-i', $object->ordered_at)->format('m-d') }}
```

Beter:

```php
// Model
protected $dates = ['ordered_at', 'created_at', 'updated_at'];
public function getSomeDateAttribute($date)
{
    return $date->format('m-d');
}

// View
{{ $object->ordered_at->toDateString() }}
{{ $object->ordered_at->some_date }}
```

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)

### **Andere goede gebruiken**

Gebruik nooit direct logica in het `routes`  bestand.

Gebruik zo min mogelijk pure PHP in Blade Templates.

Gebruik het liefst Engels als voertaal in je code.

[⬆️ Terug naar inhoudsopgave](#inhoudsopgave)
