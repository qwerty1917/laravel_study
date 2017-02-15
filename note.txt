

# 모델 생성 (마이그레이션 + 컨트롤러)
---
모델만 생성시 
`$ php artisan make:model Post`

모델 + 마이그레이션 파일
`$ php artisan make:model Post -m`

모델 + 마이그레이션 + 컨트롤러 함께 생성
`$ php artisan make:model Post -mc`

모델 + 마이그레이션 + 컨트롤러 with REST boilerplate
`$ php artisan make:model Post -mcr`

# 컨트롤러만 생성
---

컨트롤러 만들때

`$ php artisan make:controller TaskController`

커트롤러에 기본 REST boilerplate를 함께 생성
`$ php artisan make:controller TaskController -r`

# blade.php 마스터 레이아웃 분리
---

## 컨텐츠 파일 (위치: resources/views/\<anythingelse\>/)

views/layouts/master.blade.php 에 해당 컨텐츠를 합침
```php
@extends(‘layout.master’)
```

레이아웃 파일에 위치한 @yield(‘content’) 위치에 박힐 내용
```php
@section(‘content)
  <...>
@endsection
```

## 레이아웃 파일 (위치: resources/views/layouts/)

views/layouts/masterhead.blade.php 를 해당 위치에 추가함. 컨텐츠 파일과 무관
```php
@include(‘layouts.masterhead’)
```

컨텐츠 파일의 @section(‘content) ~ @endsection 부분을 넣는다.

```php
@yield(‘content’)
```

# DB저장
---
Controller측 코드 (PostController.php)
```php
Post::create([
    'title' => request('title'),
    'body' => request('body')
]);
```
여기까지만 하면 오류 발생. 사용자가 의도하지 않은 필드로 정보를 넘기는 것을 막기 위해 모델 파일에 수정 가능한 필드를 따로 명시해 주어야 한다.

Model측 코드 (Post.php)
```php
protected $fillable = ['title', 'body'];
```

# Response 간단하게 확인 (덤프)
---
```php
dd(request()->all());
```

# Validation
---
컨트롤러의 메소드 안에서
```php
$this->validate(request(), [
    'title' => 'required|max:10',
    'body'  => 'required'
]); 
```

# Controller에서 모델 id 없이 Obj 이용하기
---
## 라우터 측
`/posts/{id}` 가 아니라 `{post}` 를 parameter로 쓴다.
```php
Route::get('/posts/{post}', 'PostController@show');
```
## 컨트롤러 측
`show($id)` 가 아닌 `show(Post $post)` 로 post 모델 사용을 명시해 준다
```php
public function show(Post $post){
    return view('posts.show', compact('post'));
}
```
# 모델 생성시 주의사항
---
`class Comment extends Model`과 같은 모델 클래스는 `Model` 클래스를 상속하기 때문에 해당 클래스를 반드시 파일 안에 링크해줘야 한다.
```php
use Illuminate\Database\Eloquent\Model as Eloquent;
```
이렇게 되어 있는 것을
```php
use Illuminate\Database\Eloquent\Model;
```
이렇게 바꾸어 주자

# Tinker
---
자세히는 모르겠지만 Tinker에서 가끔 변경사항 적용 안될때도 있는듯. 테스트해가며 사용해야 할때는 `quit` 으로 종료하고 다시 `php artisan tinker`로 재시작 하여 사용한다

# 1 : M Schema
---
## 조회
### `Post.php` (1 Post : M Comment 상황일 때)
```php
class Post extends Model
{
    public function comments(){
        return $this->hasMany('App\Comment');
    }
}
```
`$post->comments`로 post에 달린 m개의 comment 모두를 찾을 수 있도록 설정한다. 라라벨이 자동으로 Comment 테이블의 한 컬럼이 Post의 FK임을 예측한다. 이때 예측 규칙은 Comment 테이블의 _id 가 붙은 컬럼명이므로 미리 Comment 테이블을 만들 때 반드시 post_id 컬럼을 포함해야 한다.

### `Comment.php`
```php
class Comment extends Model
{
    public function post(){
        return $this->belongsTo('App\Post');
    }
}
```
Comment.php 측엔 `$comment->post;`로 댓글에서 포스트를 역으로 찾을 수 있도록 설정해 둔다.
## 생성
### 방법 1
`Post.php`에 `$post->addComment(<param>)` 형태로 코멘트를 등록할수 있는 코드를 작성한다.
```php
class Post extends Model
{
    ...
    public function addComment($body){
        Comment::create([
            'body'    => $body,
            'post_id' => $this->id
        ]);
    }
}
```
`CommentController`의 `store()` 메서드에서 `Post`모델에 정의한 댓글 생성 메서드를 활용한다.
```php
class CommentController extends Controller
{
    public function store(Post $post){

        // add a comment to a post
        $post->addComment(request('body'));

        return back();
    }
}
```
# .env
---
.env 파일에 `CONSTANT_NAME=vlaue` 형태로 상수를 설정하면 다른 파일에서 `env('CONSTANT_NAME', <default_value>)`로 사용할 수 있다.
# 리다이렉트와 새로고침
---
## 현재 페이지로 리다이렉트 (새로고침)
### controller측
```php
return redirect();
```
## 다른 페이지로 리다이렉트
### controller측
```php
return redirect()->home();
```
### router측
```php
## ->name('home'); 으로 route 이름이 home임을 명시해 준다.
Route::get('/', 'PostController@index')->name('home');
```

# 비밀번호 확인 (password confirmation)
---
`<original_form>_confirmation` 으로 폼 네임을 특정하면 validation에서 `<original_form>` 에 대한 `confirmed` 조건을 바로 사용할 수 있다.
form 측
```html
<input type="password" name="password">
<input type="password" name="password_confirmation">
```
controller측
```php
$this->validate(request(), [
    ...
    'password'=>'confirmed'
]);
```
# Authentication ( Login, Register )
---
## Router
```php
Route::get('/register', 'RegistrationController@create');
Route::post('/register', 'RegistrationController@store');

Route::get('/login', 'SessionsController@create');
Route::post('/login', 'SessionsController@store');
Route::get('/logout', 'SessionsController@destroy');
```
## model
```php
class User extends Authenticatable
{
    ...
    public function posts(){
        return $this->hasMany(Post::class);
    }
}
```
## RegisterController
```php
class RegistrationController extends Controller
{
    public function create(){
        return view('sessions.create');
    }

    public function store(){
        $this->validate(request(), [
           'name'=>'required',
            'email'=>'required|email',
            'password'=>'required|confirmed'
        ]);

        // Create and save the user.
        $user = User::create([
            'name' => request('name'),
            'email' => request('email'),
            // bcrypt(<original>) hellper function 으로 암호화
            'password' => bcrypt(request('password'))
        ]);

        // Sign them in.
        auth()->login($user);

        //Redirect to the home page.
        return redirect()->home();
    }
}

```
## SessionController
```php
class SessionsController extends Controller
{

    public function  __construct(){
        $this->middleware('guest')->except('destroy');
    }

    public function create(){
        return view('sessions.create');
    }

    public function store(){
        //Attempt to auth user
        if (!auth()->attempt(request(['email', 'password']))){
            // 로그인 실패시 에러메세지 띄움
            return back()->withErrors([
                'message'=>'Please check your credentials and try again.'
            ]);
        }

        return redirect()->home();
    }

    public function destroy(){
        auth()->logout();

        return redirect()->home();
    }
}
```


## post controller ( login_required 설정 )
`auth`, `guest` 미들웨어를 이용하여 사용자나 방문자로 하여금 특정 클래스의 메소드들에 접근하는것을 제한할 수 있다. 이미들웨어들은 해당 클래스의 메소드들에 일관적용되며 몇몇 메소드를 규칙에서 제외하고 싶다면 `excpt()`를 이용한다.
```php
class PostController extends Controller
{
    public function __construct()
    {
        # index와 show를 제외한 나머지 메소드는 로그인 해야 접근 가능
        $this->middleware('auth')->except(['index', 'show']);
    }
    ...
}
```
```php
class SessionsController extends Controller
{
    public function __construct()
    {
        # guest 미들웨어를 이용하면 반대로 이미 로그인한 사용자를 차단할 수 있다.(로그인 페이지 등)
        $this->middleware('guest')->except('destroy');
    }
    ...
}
```
# GET request에 특정 파라미터가 포함되어 있는지 확인
---
```php
if(request('month')){...} // 혹은
if($month = request('month')){...} //이건 확인도 하고 변수 할당도 수행
```
# SQL 쿼리 사용
```php
// 같은 년도, 달로 묶어내기
$archives = Post::selectRaw('year(created_at) as year, monthname(created_at) as month, count(*) as published')
    ->groupBy('year', 'month')
    ->orderByRaw('min(created_at) desc')
    ->get()
    ->toArray();
```
```php
// 특정 연도, 달의 레코드 찾기 (month는 'May'를 Carbon 이용하여 5로 변환)
if ($month = request('month')){
    $posts->whereMonth('created_at', Carbon::parse($month)->month);
}

if ($year = request('year')){
    $posts->whereYear('created_at', $year);
}
```
# View composer 사용 
---
여러 뷰에서 같은 element가 사용되는 경우, 해당 element를 blade 레이아웃으로 분리한 후 해당 레이아웃이 호출될 때마다 그 레이아웃에 필요한 함수가 호출되도록 설정할 수 있다. 
## AppServiceProvider.php
```php
class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        view()->composer('layouts.sidebar', function($view){
           $view->with('archives', Post::archives());
        });
    }
    ...
}
```
위의 코드에서 `$view->with('archives', Post::archives());` 부분은 컨트롤러에서 특정 정보를 뷰로 넘겨주는 코드와 정확히 일치한다. 이렇게 컨트롤러의 일부 기능을 뷰의 일부인 레이아웃 파일과 먼저 바인딩하는 것

# 테스팅
---
테스트 코드 작성 Dir: `<myapp>/tests/Feature/`
명령어 `phpunit` (안먹힐 시 `vendor/bin/phpunit`)
`$ phpunit tests/Feature/ExampleTest.php`

```php
class ExampleTest extends TestCase
{
    public function testBasicTest()
    {
        $this->get('/')->assertSee('The Bootstrap Blog');
    }
}
```
```
$ phpunit tests/Feature/ExampleTest.php
PHPUnit 6.0.5 by Sebastian Bergmann and contributors.

.                       1 / 1 (100%)

Time: 145 ms, Memory: 18.00MB

OK (1 test, 1 assertion)
```
=> `/` 로 접근시 `The Bootstrap Blog`가 보이는지 검사한다.

## 테스트 케이스용 더미 모델 생성(or DB 시딩)
`<App>/database/ModelFactory.php`
```php
$factory->define(App\Post::class, function (Faker\Generator $faker) {
    return [
        // 실제 유저와 매칭되도록 post 생성시 user가 자동 생성되도록 함수를 할당
        'user' => function(){
            return factory(App\User::class)->create()->id;
        },
        // $faker 오브젝트의 메서드를 사용하여 적절한 더미 데이터 생성
        'title' => $faker->sentence,
        'body' => $faker->paragraphs,
    ];
});
```
`tinker`에서
`>>> factory(App\Post::class)->make();` 하나의 더미 포스트 생성 (저장X)
`>>> factory(App\Post::class)->create();` 하나의 더미 포스트 생성 및 저장
`>>> factory(App\Post::class, 50)->make();` 50개 더미 포스트 생성 (저장X)

## Unit/ExampleTest.php 수정
```php
class ExampleTest extends TestCase
{
    public function testBasicTest()
    {
        //given i have two records in the database that are posts

        //and each one is posted a month apart.
        $first = factory(Post::class)->create();
        $second = factory(Post::class)->create([
            'created_at'=>\Carbon\Carbon::now()->subMonth()
        ]);


        //when i fetch the archives
        $posts = Post::archives();


        //then the response should be in the proper format
        $this->assertCount(2, $posts);
    }
}
```
테스트를 할 때마다 새로운 레코드가 프로덕션 DB에 저장되어버리면 곤란하므로 테스트용 DB 를 따로 만든다
```
$ mysql -uroot -p
Enter password:
...
mysql> create database blog_testing
Query OK, 1 row affected (0.00 sec)
```
테스트시엔 테스트용 DB를 사용하도록 설정해준다.
`<App>/phpunit.xml`
```xml
...
<php>
    <env name="APP_ENV" value="testing"/>
    <env name="CACHE_DRIVER" value="array"/>
    <env name="SESSION_DRIVER" value="array"/>
    <env name="QUEUE_DRIVER" value="sync"/>
    <env name="DB_DATABASE" value="blog_testing"/>
</php>
...
```
테스트용 데이터베이스 마이그레이션
`.env` 에서 사용 DB를 테스트용으로 변경한 후
```
...
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=blog_testing
DB_USERNAME=root
DB_PASSWORD=
...
```
```
$ php aratisan migrate
```
`.env` 다시 원상복귀
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=blog
DB_USERNAME=root
DB_PASSWORD=
```
테스트 수행 후 테스트DB를 다시 비워주도록 설정
`Unit/ExampleTest.php`
```php
class ExampleTest extends TestCase
{
    //테스트 후 모두 롤백시킴
    use DatabaseTransactions;
    
    public function testBasicTest()
    {
    ...
    }
}
```
테스트 코드 수정
```php
class ExampleTest extends TestCase
{
    use DatabaseTransactions;

    public function testBasicTest()
    {
        
        ...
        
        //then the response should be in the proper format
        // $posts = Post::archives(); 로 불러온 $post array가 다음과 같은 format을 따라는지 확인한다.
        $this->assertEquals([
            [
                "year"      => $first->created_at->format('Y'),
                "month"     => $first->created_at->format('F'),
                "published" => 1
            ],
            [
                "year"      => $second->created_at->format('Y'),
                "month"     => $second->created_at->format('F'),
                "published" => 1
            ]
        ], $posts);
    }
}
```
#Repository
여러 컨트롤러 메소드에서 중복 작성될 수 있는 기능들을 Repository 로 묶어 생산성을 높인다.
`<App>\App\Repositories\Posts.php`
```php
<?php

namespace App\Repositories;

use App\Post;

class Posts{
    public function all(){
        // return all posts
        return Post::all();
    }

    public function find(){

    }
}
```

`PostController.php`
```php
// 레포지토리 임포트
use App\Repositories\Posts;

class PostController extends Controller
{
    ...
    // 메소드 생성시 parameter로 레포지토리 불러옴
    public function index(Posts $posts){
        $posts = $posts->all();

        return view('posts.index', compact('posts'));
    }
    ...
```
# Service provider
---
따로 찾아보기
# Mail
---
```
$ php artisan make:mail Welcome
```
`app/Mail` 디렉토리가 생성된 것을 확인할 수 있다.
해당 폴더 안에 Welcome.php 생성하고 다음과 같이 수정
```php
...
class Welcome extends Mailable
{
    ...
    public function build()
    {
        // veiws/emails/welcome.blade.php 렌더링
        return $this->view('emails.welcome');
    }
}

```
발송할 메일 blade.php 작성
```html
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h1>Welcome to Laracasts.</h1>
</body>
</html>
```
mail 발송자 정보 수정 (`config/mail.php`)
```php
...
'from' => [
    'address' => env('MAIL_FROM_ADDRESS', 'hello@example.com'),
    'name' => env('MAIL_FROM_NAME', 'Example'),
],
...
```
메일 서버 설정 (테스트용 메일 시뮬레이션 서비스인 [mailtrap](https://mailtrap.io) 사용)
```
...
MAIL_DRIVER=smtp
MAIL_HOST=mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=4933fe6a9db2c4
MAIL_PASSWORD=2b3ff35221272a
MAIL_ENCRYPTION=null
...
```
회원가입할 시 메일을 발송시킬 것이므로 `RegistrationController.php` 수정
```php
...
use App\Mail\Welcome;

class RegistrationController extends Controller
{
    ...

    public function store(){
        ...
        auth()->login($user);

        // 새로운 메일 오브젝트 만들어 발송
        \Mail::to($user)->send(new Welcome());
        ...
    }
}
```
##  Dynamic Mail
메일에 dynamic한 정보를 넘겨줄땐 해당 dynamic object를 메일 인스턴스 생성시 인자로 넘겨준다.
`RegistrationController.php` 수정
```php
\Mail::to($user)->send(new Welcome($user));
```
`Welcome.php` 수정
```php
...
// User 모델 import
use App\User;

class Welcome extends Mailable
{
    // $user 인스턴스 변수 선언
    public $user;

    // 인스턴스 생성시 $user 초기화
    public function __construct(User $user){
        $this->user = $user;
    }

    public function build()
    {
        return $this->view('emails.welcome');
    }
}
```
`welcome.blade.php` 수정
```html
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h1>Welcome to Laracasts. {{$user->name}}</h1>
</body>
</html>
```
## markdown 메일 보내기
`--markdown` 플래그를 이용하여 `views/emails/` 에 마크다운 이메일 템플릿을 함께 생성한다.
```
$ php artisan make:mail WelcomeAgain --markdown="emails.welcome-again"
```
`Http/Mail/WelcomeAgain.php`를 확인해보면 `$this->markdown(...);` 이 자동 생성되어있음을 알 수 있다.
```php
...
class WelcomeAgain extends Mailable
{
    ...
    public function build()
    {
        return $this->markdown('emails.welcome-again');
    }
}

```
`welcome-again.blade.php` 편집
```php
@component('mail::message')
# Introduction

Thanks so much for registering!

@component('mail::button', ['url' => 'https://laracasts.com'])
Start Browsing
@endcomponent

@component('mail::panel', ['url' => ''])
Lorem ipsum dolar sit amet.
@endcomponent

Thanks,<br>
{{ config('app.name') }}
@endcomponent
```
## 마크다운 메일 Customize
`rsources/vendor/mail/` 폴더에 마크다운 메일의 서식을 Publish
```
$ php artisan vendor:publish --tag=laravel-mail
```
`rsources/vendor/mail/html/themes/` 폴더에 `default.css` 가 있는데, 이게 기본 마크다운 css이다. 이 경로에 `default.css` 를 `custom.css`로 복사하고 스타일을 수정한 후 마크다운 기본 스타일시트를 아래와 같이 바꿔준다.
```php
...
'markdown' => [
    'theme' => 'custom', // default를 custom으로 변경 

    'paths' => [
        resource_path('views/vendor/mail'),
    ],
],
...
```
# Form request
---
form request 로직을 기존 controller에서 따로 빼는 기능을 제공한다.

`Http/Requests/RegistrationForm.php` 생성
```
$ php artisan make:request RegistrationForm
```

`RegistratoinController` 에서 validation 부분 잘라내고 메서드 인자로 `RegistrationForm` 오브젝트 전달
```php
class RegistrationController extends Controller
{
    ...
    public function store(RegistrationForm $form){ // 인자 추가
        // Validation
        // 잘라내기
        ...
    }
}
```

`RegistrationForm.php` 수정
```php
class RegistrationForm extends FormRequest
{
    ...
    public function rules()
    {
        // 잘라낸 Validation rules 추가
        return [
            'name'=>'required',
            'email'=>'required|email',
            'password'=>'required|confirmed'
        ];
    }
}

```
남은 로직을 빼기 위해 RegistrationForm에 메서드를 추가할 수도 있다.
```php
class RegistrationForm extends FormRequest
{
    ...
    public function persist(){
        // Create and save the user.
        $user = User::create(
            $this->only(['name', 'email', 'password'])
        );

        // Sign them in.
        auth()->login($user);

        Mail::to($user)->send(new Welcome($user));
    }
}

```

그리고 아주 깔끔해진 Controller
```php
class RegistrationController extends Controller
{
    public function create(){
        return view('registration.create');
    }

    public function store(RegistrationForm $form){

        $form->persist();

        //Redirect to the home page.
        return redirect()->home();
    }
}
```

# Session Handling & Flash Messaging (세션 & 메세지)
---
기본적인 세션의 사용법
- `session('message', 'Here is a default message');`
- `session(["message"=>'Something custom']);`

위 세션은 사용자가 로그인 할 때 갱신된다.
- `session()->flash('message', 'Thanks so much for signing up!');`

위 세션은 한번의 페이지 로드동안만 유지된다.

## 회원가입 환영 메세지 띄우기
`RegistrationController.php`
```php
public function store(RegistrationForm $form){
    $form->persist();
    
    // 세션에 플래시 메세지 추가
    session()->flash('message', 'Thanks so much for signing up!');

    return redirect()->home();
}
```
`master.blade.php`
```php
@if ($flash = session('message'))
<div id="flash-message" class="alert alert-success master-alert" role="alert">
    {{ $flash }}
</div>
@endif
```
# N:M relation pivot table (다대다 관계)
---
우선 Tag 모델과 마이그레이션 파일을 생성한다.
```
php artisan make:model Tag -m
```
`2017_02_14_015645_create_tags_table.php` 에서 마이그레이션 정보 수정
```php
class CreateTagsTable extends Migration
{
    public function up()
    {
        // Tag 테이블
        Schema::create('tags', function (Blueprint $table) {
            $table->increments('id');
            $table->string('name')->unique();
            $table->timestamps();
        });
        
        // Post-Tag relation pivot 테이블
        Schema::create('post_tag', function (Blueprint $table) {
            // post_id 와 tag_id를 정보로 가진다. (increasing primary 필요x)
            $table->integer('post_id');
            $table->integer('tag_id');
            // primary 는 두 id의 조합으로 하여 같은 외래키 조합이 중복되지 않도록 한다.
            $table->primary(['post_id', 'tag_id']);
        });

    }
    
    public function down()
    {
        // 롤백시 두 테이블 모두 드롭한다.
        Schema::dropIfExists('tags');
        Schema::dropIfExists('post_tag');
    }
}
```
마이그레이션 적용
```
$ php artisan migrate
```
`Post.php` 모델 파일에 릴레이션 정의
```php
...
class Post extends Model
{
    ...
    public function tags(){
        return $this->belongsToMany(Tag::class);
    }
}
```
`Tag.php` 모델 파일에 릴레이션 정의
```php
...
class Tag extends Model
{
    public function posts(){
        return $this->belongsToMany(Post::class);
    }
}
```
tinker 들어가서 한 포스트에 등록된 태그 모두 찾기
```php
$ php artisan tinker
>>> $post = App\Post::find(1);
...
>>> $post -> tags;
=> Illuminate\Database\Eloquent\Collection {#676
     all: [
       App\Tag {#678
         id: 1,
         name: "personnal",
         created_at: "2017-02-14 11:12:05",
         updated_at: "2017-02-14 11:12:05",
         pivot: Illuminate\Database\Eloquent\Relations\Pivot {#665
           post_id: 1,
           tag_id: 1,
         },
       },
       App\Tag {#670
         id: 2,
         name: "php",
         created_at: "2017-02-14 11:12:18",
         updated_at: "2017-02-14 11:12:18",
         pivot: Illuminate\Database\Eloquent\Relations\Pivot {#669
           post_id: 1,
           tag_id: 2,
         },
       },
     ],
   }
```
좀더 간결하게 태그 이름만 빼기
```php
>>> $post->tags->pluck('name');
=> Illuminate\Support\Collection {#660
     all: [
       "personnal",
       "php",
     ],
   }
```
한 태그가 등록된 모든 포스트 찾기
```php
>>> $tag = App\Tag::first();
=> App\Tag {#673
     id: 1,
     name: "personnal",
     created_at: "2017-02-14 11:12:05",
     updated_at: "2017-02-14 11:12:05",
   }
>>> $tag->posts;
=> Illuminate\Database\Eloquent\Collection {#674
     all: [
       App\Post {#671
         id: 1,
         user_id: 12,
         title: "My post",
         body: "...",
         created_at: "2017-01-10 06:52:10",
         updated_at: "2017-02-10 06:52:10",
         pivot: Illuminate\Database\Eloquent\Relations\Pivot {#669
           tag_id: 1,
           post_id: 1,
         },
       },
     ],
   }
```
모든 포스트 불러올 때 각 포스트에 달린 태그도 함께 불러오기 (이렇게 하면foreach 돌릴 필요 없음)
```php
>>> App\Post::with('tags')->get();
```
모든 태그 불러올 때 각 태그가 달린 포스트들도 함께 불러오기
```php
>>> App\Tag::with('posts')->get();
```
태그 이름으로 찾기
```php
>>> $tag = App\Tag::where('name', 'personal')->first();
```
포스트에 태그 등록하기
```php
>>> $post = App\Post::first(); // 포스트 하나 부르고
>>> $tag = App\Tag::where('name', 'personal')->first(); // 태그 하나 부르고
>>> $post->tags()->attach($tag); // attach() 사용
=> null
```
# 태그로 포스트 정렬하기
---
`TagsController` 생성
```
$ php artisan make:controller TagsController
```
url에서 태그 이름으로 접근 가능하도록 (ex: /posts/tags/personal) Tag.php 모델에 다음과 같은 메서드 추가
```php
...
class Tag extends Model
{
    ...
    public function getRouteKeyName(){
        return 'name';
    }
}
```
## 사이드바에 태그 목록을 띄워보자
`AppServiceProvider.php`
```php
...
public function boot()
{
    view()->composer('layouts.sidebar', function($view){
       $view->with('archives', Post::archives());
       $view->with('tags', Tag::pluck('name')); // 추가
    });
}
...
```
사이드바 view 에 태그 블록 추가 (`sidebar.blade.php`)
```php
<div class="col-sm-3 offset-sm-1 blog-sidebar">
    ...
    <div class="sidebar-module">
        <h4>Tags</h4>
        <ol class="list-unstyled">
            @foreach($tags as $tag)
                <li>
                    <a href="/posts/tags/{{ $tag }}">
                        {{ $tag }}</a>
                </li>
            @endforeach
        </ol>
    </div>
    ...
</div><!-- /.blog-sidebar -->
```
어느 포스트에도 할당되지 않은 태그는 다음과 같이 뺄 수 있다.
```php
>>> App\Tag::has('posts')->pluck('name'); // post를 가진 태그만 반환
```
따라서 `AppServiceProvider.php` 아래와 같이 다시 수정
```php
...
public function boot()
{
    view()->composer('layouts.sidebar', function($view){
       $view->with('archives', Post::archives());
       $view->with('tags', Tag::has('posts')->pluck('name')); // has('posts') 추가
    });
}
...
```
어차피 컨트롤러에서 템플릿 렌더링 하는 부분과 동일하므로 아래처럼 깔끔하게 수정할 수 있다.
```php
public function boot()
{
    view()->composer('layouts.sidebar', function($view){
        $archives = Post::archives();
        $tags = Tag::has('posts')->pluck('name');

        $view->with(compact('archives', 'tags'));
    });
}
```
마지막으로 post show 화면에 태그 링크 추가 (`show.blade.php`)
```php
...
@if(count($post->tags))
<ul>
    @foreach($post->tags as $tag)
        <li>
            <a href="/posts/tags/{{$tag->name}}">{{$tag->name}}</a></li>
    @endforeach
</ul>
@endif
...
```
# Eventing 이벤트
---
이렇게 이벤트와 리스너 파일을 직접 만들어 줄수도 있지만
```
php artisan make:event SomeEvent
php artisan make:listener SendNotification --event="SomeEvent"
```
`EventServiceProvider.php`에 원하는 이벤트와 리스너를 등록하고 한번에 생성 시키는것이 편하다. 직접 파일을 만든 경우에도 여기에 이벤트와 리스너 파일을 꼭 등록시켜줘야 한다.
```php
class EventServiceProvider extends ServiceProvider
{
    ...
    // 생성할 이벤트와 리스너. 경로의 마지막 클래스 이름만 원하는 것으로 해주면 된다.
    protected $listen = [
        'App\Events\Forum\ThreadCreated' => [
            'App\Listeners\Forum\NotifySubscribers',
        ],
    ];
    ...
}
```
이벤트 및 리스너 생성 커맨드
```
$ php artisan event:generate
```
`ThreadCreated.php` 이벤트 파일 작성
```php
class ThreadCreated
{
    ...
    public $thread; // 이벤트에 저장할 데이터 속성

    public function __construct($thread) // 이벤트 발생시 데이터 오브젝트 인자로 받음
    {
        $this->thread = $thread; // 이벤트 내에 오브젝트 저장
    }
    ...
}

```
`NotifySubscribers.php` 수정
```php
class NotifySubscribers
{
    ...
    public function handle(ThreadCreated $event)
    {
        // 리스너의 동작 정의
        var_dump($event->thread['name'].' was published to the forum');
    }
}
```
이벤트 테스트
```php
$ php artisan tinker
>>> event(new App\Events\Forum\ThreadCreated(['name'=>'Some new thread']));
...
string(42) "Some new thread was published to the forum"
=> [
     null,
   ]
```
## event:generate 안쓰고 직접 생성
```
$ php artisan make:listener CheckForSpam --event="ThreadCreated"
```
```php
...
namespace App\Listeners\Forum; // 현재 경로에 맞게 수정
...
use App\Events\Forum\ThreadCreated; // 임포트 디렉터리 알맞게 수정
...
class CheckForSpam
{
    ...
    public function handle(ThreadCreated $event)
    {
        // 리스너 액션 수정
        var_dump('Checking for spam');
    }
}
```
`EventServiceProvider.php` 수정
```php
protected $listen = [
    'App\Events\Forum\ThreadCreated' => [
        'App\Listeners\Forum\NotifySubscribers',
        'App\Listeners\Forum\CheckForSpam', // 실제 경로에 맞추어 수정
    ],
];
```
이벤트 실행
```php
>>> event(new App\Events\Forum\ThreadCreated(['name'=>'Some new thread']));
...
string(17) "Checking for spam"
=> [
     null,
     null,
   ]
```