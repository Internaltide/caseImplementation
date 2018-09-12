# Adldap Package Case Study - Login Auth Via Adldap

>&nbsp;<br/>
> 案例名稱：**CS0001-Soft-Laravel-Authenticate-LdapLoginAuth**<br/>
> 應用類型：**Authenticate**<br/>
> 使用套件：**Adladp2-Laravel**<br/>
> 套件來源：https://github.com/Adldap2/Adldap2-Laravel<br/>
> 套件版本：4.0<br/>
> 框架版本：Laravel 5.6<br/>
> 案例目標：讓Laravel應用可藉由LDAP Server做使用者登入驗證，並整合於Laravel原生驗證結構中<br/>
>&nbsp;<br/><br/>

> ## Laravel Authenticate 回顧
### Laravel內建的驗證路由
Auth::routes()
```
// Authentication Routes...
$this->get('login', 'Auth\LoginController@showLoginForm')->name('login');
$this->post('login', 'Auth\LoginController@login');
$this->post('logout', 'Auth\LoginController@logout')->name('logout');

// Registration Routes...
$this->get('register', 'Auth\RegisterController@showRegistrationForm')->name('register');
$this->post('register', 'Auth\RegisterController@register');

// Password Reset Routes...
$this->get('password/reset', 'Auth\ForgotPasswordController@showLinkRequestForm')->name('password.request');
$this->post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail')->name('password.email');
$this->get('password/reset/{token}', 'Auth\ResetPasswordController@showResetForm')->name('password.reset');
$this->post('password/reset', 'Auth\ResetPasswordController@reset');
```

### Laravel 登入驗證結構簡述
#### 控制器
透過 Trait **Illuminate\Foundation\Auth\AuthenticatesUsers** 來定義需要控制器方法，<br/>
AuthenticatesUsers Trait 中跟跳轉、登入失敗控制相關的方法又各自獨立到下列兩個Traits<br/>

- **Illuminate\Foundation\Auth\RedirectsUsers**
- **Illuminate\Foundation\Auth\ThrottlesLogins**<br/>

兩個獨立出來的Trait，也可以更方便地在未來直接引入到客製的控制器中來擴充功能。

#### Guard - 提供用戶端認證的互動方法
官網是定義為每個請求內如何對使用者進行身分驗證，但認證部分主要還是抽象的互動，<br/>
實際驗證邏輯仍是由更底層類別實作。另外還有驗證狀態的維護等。<br/>
該類別實作了 **Illuminate\Contracts\Auth\Guard** 這個介面，所有客製的Guard也都需要<br/>
實作該介面，並透過服務提供者的boot方法進行擴充綁定，才能透過設定檔來<br/>
指定使用並整合進現有的Laravel驗證架構中，否則預設只支援session、token兩種。<br/>
```
Auth::extend('jwt', function ($app, $name, array $config) {
    // Return an instance of Illuminate\Contracts\Auth\Guard...
    return new JwtGuard(Auth::createUserProvider($config['provider']));
});
```

#### User Provider - 提供認證邏輯實作及取得User模型
此物件由前述的Guard來合成使用，主要設計用來取得欲驗證的使用者模型並提供驗證方法validateCredentials<br/>
供守衛Guard的attempt方法來調用，該類別還實作了 **Illuminate\Contracts\Auth\UserProvider** 這個介面。<br/>
所有客製的User Provider也都需要實作該介面，並透過服務提供者的boot方法進行擴充綁定，才能透過設定檔來<br/>
指定使用並整合進現有的Laravel驗證架構中，否則預設只支援database、eloquent兩種。<br/>
```
Auth::provider('riak', function ($app, array $config) {
    // Return an instance of Illuminate\Contracts\Auth\UserProvider...
    return new RiakUserProvider($app->make('riak.connection'));
});
```

#### User 模型 - 直接與底層儲存溝通，取得用戶資訊
此物件由前述的User Provider來合成使用，預設就是一個eloquent模型，可透過ORM機制存取使用者資料。<br/>
該User模型類別還實作了 **Illuminate\Contracts\Auth\Authenticatable** 這個介面，提供User Provider所需<br/>
的互動方法。所有客製的User模型也都需要實作該介面，才能正確整合進現有的Laravel驗證架構中。<br/>

另外，該模型亦實作下列兩個介面，但與驗證程序無關，主要是額外提供權限控制等功能。<br/>

 - **Illuminate\Contracts\Auth\Access\Authorizable**
 - **Illuminate\Contracts\Auth\CanResetPassword**<br/>

 並分別透過下面兩個trait來提供實作<br/>

 - **Illuminate\Foundation\Auth\Access\Authorizable**
 - **Illuminate\Auth\Passwords\CanResetPassword**<br/><br/>

> ## LDAP 身分認證實作案例
### 安裝套件
#### 安裝指令
```
composer require adldap2/adldap2-laravel
```
#### 套件發佈
Laravel5.6的Service Provider已有Discovery機制，Adldap-laravel亦已支援。<br/>
所已無須再手動到config/app註冊，框架會自動發現並載入。使用前僅需要發佈套件設定檔<br/>
至應用程式內，即可開始進行配置及使用
```
php artisan vendor:publish --tag="adldap"
```
#### 注意事項
要知道的是Adldap套件本身會進行LDAP延伸的檢查，所以要注意，若是在Laradock環境下<br/>
，workspace裡頭也要需有LDAP延伸，否則在workspace下使用composer安裝套件時就會安裝失敗。

### 使用的Guard
此案例沿用既有的認證抽象互動與驗證狀態維護方式，僅是需要提供不同的Credential Validate實作<br/>
，亦即改成LDAP驗證實作，所以沿用Laravel內建的sessionGuard即可。

### 使用的User Provider
Adladp套件已內建好UserProvider驅動，並已在AdldapAuthServiceProvider中進行註冊，<br/>
所以可以直接在config/auth裡指定使用套件的provider driver。不過，套件預設提供了兩種UserProvider，<br/>
實際使用那一個仍須到config/adldap_auth去設定。<br/>

 - **Adldap\Laravel\Auth\DatabaseUserProvider::class**<br/>
實作兩段式驗證，先定位LDAP的使用者，再進行LDAP Bind驗證。另外，該UserProvider<br/>
於驗證成功後會將LDAP User的資訊也同步到本地資料庫者，也因此仍需要建立User Table。<br/>

 - **Adldap\Laravel\Auth\NoDatabaseUserProvider::class**<br/>
亦是實作兩段式驗證，但沒有資料同步功能，所以也無須準備本地的User Table。<br/>
該Provider使用套件內建的User模型，並非典型的Eloquent模型，config/app內可以忽略model的設定。<br/>
**不過，這個UserProvider有很多的問題，看來並不是很穩定。目前，使用上雖然可成功提供身份的驗證，**<br/>
**但在之後的Request並無法取得Authenticated User，導致無法在之後的請求進行判斷是否已有成功的登入。**<br/>
**筆者起初是要使用這個Provider，但過程中痛苦體驗太多，遂只能放棄並改用DatabaseUserProvider。**<br/>
**建議，套件4.0以前的版本，不要使用NoDatabaseProvider。**

### 使用的User模型
#### 資料模型
 - NoDatabaseUserProvider
   此Provider會使用套件自己的使用者模型，並與LDAP伺服器溝通取得所需資料。<br/>
   所以不需要特別指定使用哪一個使用者模型，甚至設定檔也可以忽略掉model一項。<br/>

 - DatabaseUserProvider
    除了認證邏輯改用LDAP Bind外，依然是透過原本的Eloquent User Model<br/>
    與本地資料庫溝通並取得所需要的使用者資訊。所以可使用Laravel預先建立<br/>
    好的User模型，再依專案的需求並按Laravel規定進行配置與覆寫。<br/>

#### 客製部分
擷取登入用戶所屬網域並同步至本地資料庫，但因LDAP並無可直接取用的網域屬性，需透過DN屬性進行轉換。<br/>
 - 透過套件自身的設定檔設定將DN屬性同步到本地資料表的domain欄位，設定方式詳見稍後的設定說明區塊<br/>
 - 建立模型修改器Mutators，修改器內容則執行DN轉換網域的邏輯實作

##### Mutator Method
 ```
 ...
 use App\Traits\LdapAttrParserTrait;
 ...

  /**
     * Set the user's domain.
     *
     * @param  string  $value
     * @return void
     */
    public function setUserDomainAttribute($dn)
    {
        $domain = $this->fetchDomainFromDN($dn);

        $this->attributes['user_domain'] = ( empty($domain) ) ? null:$domain;
    }
 ```
 ##### Trait
 ```
 trait LdapAttrParserTrait
{
     /**
     * Convert Distinguished Name to an Domain string
     *
     * @return string
     */
    public function fetchDomainFromDN($dn)
    {
        if( empty($dn) ) return '';

        $sepratedomain = array();
        $sepratedn = explode(',',$dn);
        foreach ($sepratedn as $key => $compoent) {
            if( preg_match('/^(dc=){1}[a-zA-Z0-9\-]+/', $compoent) ){
                $sepratedomain[] = explode('=', $compoent)[1];
            }
        }

        return strtolower( implode('.', $sepratedomain) );
    }
}
 ```

### 資料庫遷移設定
```
/**
 * Run the migrations.
 *
 * @return void
 */
public function up()
{
    Schema::connection('migrations')->create('users', function (Blueprint $table) {
        // Define Table Option
        $table->engine = 'InnoDB';
        $table->charset = 'utf8';
        $table->collation = 'utf8_general_ci';

        // Define Primary Key
        $table->unsignedInteger('id')->autoIncrement();

        // Define Table Column
        $table->string('user_id', 30)->comment('LDAP Identifier');
        $table->string('user_name', 100)->comment('LDAP Displayname');
        $table->string('login',100)->comment('LDAP Accountname');
        $table->string('password', 255)->comment('LDAP Password');
        $table->string('user_domain',50)->comment('DOMAIN');
        $table->string('email', 255)->comment('LDAP Mail');
        $table->unsignedTinyInteger('active')->default(1)->comment('啟用與否');
        $table->rememberToken();

        // Create Index
        $table->unique('user_id');
        $table->index('email');
    });
}
```

### 設定
#### config/auth
套件本身就是依Laravel既有認證結構來實作，所以按原設定方式來啟用LDAP身份認證即可<br/>
providers driver 請改為 **adldap**
```
 'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'ldap',
        ],

        ...
    ],
```
```
 'providers' => [
        'ldap' => [
            'driver' => 'adldap',
            'model' => App\User::class, // 若用的是NoDatabaseUserProvider則可忽略此項，因為套件會用自己的模型
        ],

        'users' => [
            'driver' => 'eloquent',
            'model' => App\User::class,
        ],

        ...
    ],
```
#### config/adldap
維持adldap運作所需配置項目。該套件設定檔內多數都使用了 **env(ConfigName, DefaultValue)** 這樣的形式，<br/>
直接複製到環境設定檔，可以方便不同環境進行覆寫。主要僅列重要或異動項目，其餘不詳列。
##### 可複製到 **.env** 的設定項目
```
// 這邊幾乎都屬於connection_setting區塊內的設定項
ADLDAP_AUTO_CONNECT=true                    // 是否自動連接LDAP，會使用預設設定
ADLDAP_ACCOUNT_PREFIX=""                    // 驗證帳號名稱前綴
ADLDAP_ACCOUNT_SUFFIX=""                    // 驗證帳號名稱後綴
ADLDAP_CONTROLLERS=192.168.1.123            // LDAP 伺服器FQDN或IP，可多個並以空白隔開
ADLDAP_PORT=389
ADLDAP_TIMEOUT=5                            // 等待LDAP回應時間
ADLDAP_BASEDN="dc=whatsoft,dc=com"          // LDAP Base DN Attribute
ADLDAP_ADMIN_USERNAME="cn=admin,dc=whatsoft,dc=com"     // 管理員帳號，OpenLDAP需用完整DN才能驗證管理帳號
ADLDAP_ADMIN_PASSWORD=************          // 管理員密碼
ADLDAP_ADMIN_ACCOUNT_PREFIX=""              // 管理帳號名稱前綴
ADLDAP_ADMIN_ACCOUNT_SUFFIX=""              // 管理帳號名稱後綴
ADLDAP_USE_SSL=false                        // 使用SSL加密連線
ADLDAP_USE_TLS=false                        // 使用TLS加密連線
```
##### 其它異動項目
```
// LDAP Schema定義類別，預設提供ActiveDirectory::class與OpenLDAP::class兩種
'schema' => Adldap\Schemas\OpenLDAP::class
```
PS. 實作發現，Schema這塊在使用NoDatabaseUserProvider時影響較大。<br/><br/>
**以AuthIdentifier為例( 即認證成功後，保存到session驗證狀態變數內的值 )**
 - **DatabaseUserProvider** 使用的是Laravel的Eloquent User模型，其getAuthIdentifierName方法定義的<br/>
   即為本地資料庫User表的主鍵名稱，Eloquent模型設定好就沒問題。<br/>
 - **NoDatabaseUserProvider** 使用的是套件內建模型，其getAuthIdentifierName方法內定是取用<br/>
   objectGUID，而實際對應的GUID名稱就是由schema項設定的類別來決定。如果環境的LDAP使用了自訂<br/>
   的Schema，這邊就必須進行調整，否則就會出問題。

##### 補充事項
 - 連接用管理帳號: 參考網路其他意見，有提到不見得非要提供管理員帳密。<br/>
   但提供的帳號必須要有一定的存取權限，此部分未測試。
 - 加密連線預設並未啟用，但若有提供變更密碼等侵入式行為，為了安全，建議要啟用

#### config/adldap_auth
維持adldap身份驗證所需的配置。同樣的，使用了 **env(ConfigName, DefaultValue)** 形式的設定項<br/>
亦可複製到環境設定檔，方便覆寫。
##### 可複製到 **.env** 的設定項目
```
// 設定使用的LDAP連接配置
ADLDAP_CONNECTION=default

// 設定即時同步密碼
ADLDAP_PASSWORD_SYNC=true

// 不啟用驗證失敗轉本地驗證
ADLDAP_LOGIN_FALLBACK=false
```
##### provider
設定使用的UserProvider
```
'provider' => Adldap\Laravel\Auth\DatabaseUserProvider::class,
```
##### scopes
因為用的是OpenLDAP，所以按指示將scopes設定改為Adldap\Laravel\Scopes\UidScope::class，<br/>
取消預設使用的Adldap\Laravel\Scopes\UpnScope::class。
```
 'scopes' => [

        // Only allows users with a user principal name to authenticate.
        // Remove this if you're using OpenLDAP.
        // Adldap\Laravel\Scopes\UpnScope::class,

        // Only allows users with a uid to authenticate.
        // Uncomment if you're using OpenLDAP.
        Adldap\Laravel\Scopes\UidScope::class,

    ],
```
##### usernames
LDAP驗證分成兩個階段，先用discover屬性到LDAP Server找出使用者，再用authenticate屬性進行綁定驗證。<br/>
此區塊即定義LDAP驗證過程中各步驟使用的欄位。
```
 'usernames' => [

    'ldap' => [
        'discover' => 'uid',         // Discover階段使用的LDAP屬性
        'authenticate' => 'dn',   // Bind Authenticate階段使用的LDAP屬性
     ],

    'eloquent' => 'login',        // 定義本地資料庫使用的驗證欄位(僅在DatabaseUserProvider有效)
    ...
    ..
    .

 ],
```
##### passwords
開啟密碼即時同步，隨時將驗證成功的密碼同步到本地資料庫<br/>
 - true: 用bcrypt()對真實密碼雜湊演算後，儲存到本地資料庫。後續可使用本地資料庫進行驗證。
 - false: 隨機產生16位元字串並執行雜湊演算後再儲存，後續無法支援本地資料庫驗證。
```
 'sync' => env('ADLDAP_PASSWORD_SYNC', false), //這邊保持預設關閉，複製一份到.env由環境設定值覆寫
```
設置本地資料庫密碼欄位名稱
```
'column' => 'password',
```
##### login fallback
當LDAP驗證失敗，是否改用本地資料庫驗證。注意!! 仍需LDAP處於正常連線狀態下，<br/>
始有效，否則仍會直接吐出無法聯絡LDAP的錯誤。
```
 'login_fallback' => env('ADLDAP_LOGIN_FALLBACK', false), //這邊保持預設關閉，複製一份到.env由環境設定值覆寫
```
##### sync attributes
設定資料同步所需的欄位Mapping。Key表示本地資料庫欄位名稱；Value則表示LDAP屬性
```
 'sync_attributes' => [
    'user_id' => 'uidnumber',
    'login' => 'uid',
    'email' => 'mail',
    'user_domain' => 'dn',
    'user_name' => 'displayname',
],
```
##### logging
套件提供了相關LDAP驗證事件的Log寫入，預設開啟。另外，開啟LOG寫入時，還可透過註解方式予以<br/>
關閉特定事件的LOG寫入。
```
 'logging' => [

    'enabled' => true,

    'events' => [

        //\Adldap\Laravel\Events\Importing::class => \Adldap\Laravel\Listeners\LogImport::class,
        //\Adldap\Laravel\Events\Synchronized::class => \Adldap\Laravel\Listeners\LogSynchronized::class,
        //\Adldap\Laravel\Events\Synchronizing::class => \Adldap\Laravel\Listeners\LogSynchronizing::class,
        \Adldap\Laravel\Events\Authenticated::class => \Adldap\Laravel\Listeners\LogAuthenticated::class,
        \Adldap\Laravel\Events\Authenticating::class => \Adldap\Laravel\Listeners\LogAuthentication::class,
        ...
        ..
        .

    ],
```

### 驗證方式
符合Larvel的原生驗證架構，大致用法都相同。但注意，此時attemp方法不再支援<br/>
指定額外的驗證欄位。例如Auth::attempt(['email' => $email, 'password' => $password, **'active' => 1**])<br/>
中的active於此處就沒有效用。這也是本案例最後再透過其它方式處理Active欄位功能的原因。
```
// 進行登入驗證
Auth::attempt($credentials);

// 取得已驗證使用者
Auth::user();
```

### 其它手動實作
#### 路由
```
// 可直接沿用Auth::routes()就好，但筆者想用自己的路由命名，所以重新定義
Route::get('/login', 'LoginController@showLoginForm')->name('loginform');
Route::post('/login', 'LoginController@login')->name('login');
Route::get('/logout', 'LoginController@logout')->name('logout');

// 為所有後台路由定義使用內建的中介層auth，確保僅有通過認證的使用者可以使用後台功能
Route::group(['prefix' => 'admin', 'middleware' => ['auth']], function(){
    // 所以後台路由設定
});
```
#### 控制器
沿用內建的控制器進行修改即可，比較方便，可以直接沿用原本就建好的RedirectUsers、ThrottlesLogins等功能。
```
<?php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Foundation\Auth\AuthenticatesUsers;
use App\Services\Auth\LimitLoginService;

class LoginController extends Controller
{
    use AuthenticatesUsers;

    protected $guard;
    private $limitLoginSrv;

    public function __construct(LimitLoginService $limitLoginSrv)
    {
        $this->guard = Auth::guard();
        $this->limitLoginSrv = $limitLoginSrv;
    }

    /**
     * 重寫登入表單頁的Render方法，主要
     * 1. 加上檢查避免已登入者再一次進入登入頁並重新執行登入作業
     * 2. 統一控制表單內帳號欄位與密碼欄位的name
     */
    public function showLoginForm()
    {
        if (Auth::check()) {
            return redirect()->intended($this->redirectPath());
        }

        return view('themes/default/login', [
            'username' => $this->username(),
            'password' => $this->password()
        ]);
    }

    // 重寫表單驗證規則方法，主要套用欄位名稱控制方法
    protected function validateLogin(Request $request)
    {
        $this->validate($request, [
            $this->username() => 'required | string',
            $this->password() => 'required | string',
        ]);
    }

    /**
     * 重寫attemptLogin方法，主要
     * 1. 調用attempt前，先透過LimitLogin Service Class的 ifActiveLoginUser判斷使用者是否被停用，
     *     限制經啟用的User才能夠登入
     */
    protected function attemptLogin(Request $request)
    {
        $credentials = $this->credentials($request);

        return ( $this->limitLoginSrv->ifActiveLoginUser($credentials[$this->username()], true) )
            ? $this->guard->attempt($credentials):false;
    }

    // 覆寫預設使用的驗證帳號欄位名稱
    public function username()
    {
        return 'login';
    }

    // 新增控制密碼欄位名稱
    private function password()
    {
        return 'password';
    }

    // 重寫credentials產生方法，主要套用欄位名稱控制方法
    protected function credentials(Request $request)
    {
        return $request->only($this->username(), $this->password());
    }

    // 覆寫登入後跳轉位置
    protected function redirectTo()
    {
        return '/';
    }

    protected function guard()
    {
        return $this->guard;
    }
}
```

#### 客製服務提供Active控制功能
Adldap套件內的attempt驗證方法並無法支援指定額外驗證條件，所以，<br/>
特定新增limitLoginSrv類別，定義額外的登入限制條件。然後在調用驗證方法<br/>
方法前先行判斷。
```
<?php

namespace App\Services\Auth;

use Illuminate\Http\Request;
use App\Repositories\User\UserRepo;

class LimitLoginService
{
    private $userRepo;

    public function __construct(UserRepo $userRepo)
    {
        $this->userRepo = $userRepo;
    }

    /**
     * 透過登入名稱到本地資料庫檢查對應使用者資料的active flag是否為啟用
     * $asTrueWhenNotExist用來定義是否將不存在於本地資料庫的新登入用戶視為啟用
     */
    public function ifActiveLoginUser($loginName, $asTrueWhenNotExist=false)
    {
        $userAttemptLogin = $this->userRepo->getUserByLogin($loginName);

        if( is_null($userAttemptLogin) ) return $asTrueWhenNotExist;

        return ($userAttemptLogin->active==1) ? true:false;
    }
}
```
PS. **$userRepo**為一控制透過user model取得各類型資料集合的類別，詳細實作不多贅述

### 其他注意事項
**表單欄位的隱性限制**<br/>
套件的實作方式，使得表單欄位命名會影響驗證的正常運作，但官方文件並沒有足夠清楚的提示或說明。<br/>
這邊特別列出來，以供未來參考，避免又再度卡住。以下為參考規則：<br/>

 - **DatabaseUserProvider** 使用此Provider時，表單帳號的name必須與資料庫帳戶欄位名稱相同
 - **NoDatabaseUserProvider** 使用此Provider時，表單帳號的name必須與Discover屬性名稱相同

 如果違反了上述原則，Undefined index這種極度不直覺的錯誤訊息就會浮現。<br/>
 其實，用資料庫欄位作為表單欄位名稱再正常不過了。只是NoDatabaseUserProvider的情境下<br/>
 就很容易糊塗，加上<br/>
  - 拋出來的錯誤訊息並不是很符合實際狀況
  - 控制器的username方法也須同步改用與Discover屬性相同的名稱，不但違背原始設計，<br/>
    當要改回使用典型驗證時，這邊就得再改一次<br/>

**所以，設計上有似乎有點問題**

**表單密碼欄位名稱**<br/>
表單密碼欄位強制要用password，不然會出現 **A password must be specified.** 的錯誤訊息<br/>
若是在產品維護期中才要改用LDAP驗證，加上密碼欄位名稱又不是password，那用此套件就會有比較大的麻煩。<br/>

**登入狀態檢查失效**<br/>
這是使用NoDatabaseUserProvider才會出現的問題，也是何以最後實作會改採用DatabaseUserProvider的原因。<br/>
它是壓垮駱駝的最後一根草。<br/>

筆者約略看了相關的程式碼，發現原本應直接存到session的AuthIdentifier，會先經過一些轉換<br/>
的動作後才存入session。本來這也不是什麼大問題，但問題就是後續Request中，當要利用該<br/>
AuthIdentifier取得已驗證使用者時，卻出現AuthIdentifier無法被反轉換回原先的AuthIdentifier，<br/>
故無法正確取得已驗證的用戶模型，進而導致登入狀態檢查失效等相關問題。