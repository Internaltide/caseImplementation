# Laravel Package Development - Compoent Injection

>&nbsp;<br/>
> 案例名稱：**CS0001-Soft-Laravel-Package-ComponentInjection**<br/>
> 應用類型：**Package Development**<br/>
> 使用套件：**無**<br/>
> 套件來源：<br/>
> 套件版本：<br/>
> 框架版本：Laravel 5.6<br/>
> 案例目標：進行一次套件開發以了解開發的標準流程，以元件注入服務為例<br/>
>&nbsp;<br/><br/>

> ## 套件開發環境
### 1. 創建 Laravel 專案
建立一個獨立的專案環境，往後的套件開發都放在同一個專案裡，方便管理
```
composer create-project laravel/laravel Packages 5.6.* --prefer-dist
```
PS. 記得!! 若是在使用 Laradock 的開發環境，需先進到 workspace 工作環境裡頭才能使用 composer 指令。

### 2. 調整目錄權限
因為是自己的開發環境，簡單將目錄權限設為777即可(線上或安全性需求高時不建議這樣做)
```
cd Packages
chmod -R 777 storage
chmod -R 777 bootstrap/cache
```

### 3. 建立虛擬主機
若有輸出到瀏覽器上進行簡易偵錯的需求或套件本身就是含路由、視圖之完整功能套件時，<br/>
可以順便將虛擬主機建置起來。這裡會以 Laradock 的 nginx 環境為例進行配置。

#### Laradock nginx 配置
```
若目前是在 workspace 工作環境，請先離開
exit

建立虛擬主機設定檔
cp /home/laradep/Laradock/nginx/sites/laravel.com.example /home/laradep/Laradock/nginx/sites/laravel.Packages.conf

設定檔異動項目
server_name packages.whatsoft.com
root /var/www/Packages/public

重啟容器
docker-compose down
docker-compose up -d nginx mysql memcached
```

#### Windows DNS 解析設定
```
加入
{DOCKER 主機IP} {連結網址}
Ex. 192.168.1.124 packages.whatsoft.com
```

### 4. 專案目錄架構
首先，專案根目錄下建立 packages 放置所有套件原始碼。並以該目錄為根依序建立第二層 Vendor 目錄、第三層 Package 目錄 及第四層 src 目錄。<br/>
此案例將建置 **packages/internaltide/componentInjection/src**。<br/>
而往後的版控將以各個套件目錄為基準，一個套件對應一個 Github Respository。
另外，下面幾個目錄為此範例套件所需要，非必要：
 - 套件目錄下建立 config 目錄，放置套件的設定檔。
 - src 目錄下建立 Components 目錄，放置各種內建元件的類別定義。
 - src 目錄下建立 Views 目錄，放置內建元件用視圖
```
- Packages
  + app
  + bootstrap
  + config
  + database
  - packages
    - {vendor}
      - {package}
        + config
        - src
          + Components
          + Views
  + public
  + resources
  + routes
  + storage
  + tests
  +vendor
  ...
  ..
  .
```

> ## 套件初始配置
### 1. 建立 composer.json
為了定義套件的依賴與其命名空間，每個套件都必須含有一份 composer.json 檔案。<br/>
此例僅依需要建立相關配置，更詳細的設定項目可以參考：<br/>
https://getcomposer.ycnets.com/doc/01-basic-usage.md
```
進入到套件主目錄
cd packages/internaltide/componentInjection

composer init
```
PS. 指令 composer 初始化 composer.json 時會詢問許多問題，多數可以直接 Enter 套用預設值，快速完成建立後再編輯修改即可。若已有其他開發好的套件存在，也可以直接複製一份
過來修改，如此即不用透過 composer 指令。

#### 套件 Composer.json 內容
```
{
    "name": "internaltide/component-injection",
    "description": "The component injection service",
    "type": "library",
    "authors": [
        {
            "name": "Darren Ting",
            "email": "tingchihua@gmail.com"
        }
    ],
    "require": {},
    "autoload": {
        "Internaltide\\ComponentInjection\\": "src/"
    }
}
```

### 2. 調整 PSR-4 自動載入器
#### Laravel 專案本身
預設，Laravel 的 PSR-4 映射並無法自動載入新增的套件檔案，為方便測試，我們可以自行為
該 PSR-4 自動載入器新增套件的映射配置。
```
{
    "name": "laravel/laravel",
    "description": "The Laravel Framework.",
    "keywords": ["framework", "laravel"],
    (略)
    "autoload": {
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/",
            "Internaltide\\ComponentInjection\\": "packages/internaltide/componentInjection/src/"  //新增此行配置
        }
    },
    (略)
}
```
PS. PSR-4是種命名空間與檔案路徑的映射標準，以上述配置為例，當命名空間為<br/>
**Internaltide\ComponentInjection\\** CertainClass<br/>
其代表著映射到<br/>
**packages/internaltide/componentInjection/src/** 下的 CertainClass 類別

##### 重新產生 autoload 映射文件
一旦異動了 autoload 區塊的配置，記得使用 **composer dump-autoload** 來重建 vendor/composer/下的映射文件，這樣應用程式才能正確載入套件檔案。

#### 套件本身
套件發佈後，不可能也要求套件使用者在他們的專案調整 autoload 的路徑映射，所以套件本身的 composer.json 亦需要設定自身的映射方式。
```
(略)
"require": {},
"autoload": {
    "psr-4": {
        "Internaltide\\ComponentInjection\\": "src/"
    }
}
```

### 3. 建立服務提供者
#### 指令創建服務提供者
```
php artisan make:provider ComponentInjectionServiceProvider
```

該服務提供者是作為其他 Laravel 應用與套件的連結，將其移往套件的 src 目錄下<br/>
(檔案路徑變更後，務必記得更正檔案的命名空間並符合先前設置的 PSR-4 映射方式)
```
mv app/Providers/ComponentInjectionServiceProvider.php  packages/internaltide/componentInjection/src/
```

#### 緩載服務提供者
若提供者僅用於註冊綁定到服務容器，可以選擇延緩註冊，直到真正需要時才載入，而非一開始就載入，可以增進效能。
```
服務提供者加入 defer 屬性，並設定為 true
protected $defer = true;

服務提供者加入 provides 實作，用以後續取得提供者所在服務容器裡綁定的名稱。
public function provides()
{
    return [
        'component.factory'
    ];
}
```
PS. 注意!! 在套件提供視圖供使用者繼承使用的情境下，若設定了緩載可能造成用戶找不到你的套件視圖，即便你正確使用了 loadViewsFrom 對套件視圖目錄進行了註冊。因為套件根本還沒載入，自然你的 loadViewsFrom 也不會被調用。此時，你只能選擇將套件視圖發佈到使用者的視圖目錄或是考慮不要使用緩載的設定。

> ## 套件功能實作
### 1. 套件建置動機
Laravel Blade 本身雖已提供類似的 Component 功能，也提供了動態參數嵌入。但是當元件重複使用程度高時，仍導致多個控制器都需重複實作 Component 資料取得的邏輯或重複調用相同的資料函數。為了收斂實作及調用，也簡化控制器的職責，故額外建置元件注入的服務，讓元件資料取得邏輯收斂於各元件類別中。最後，透過工廠方法予以返回完整的元件原始碼。
```
// Laravel Blade Component Example

// AppServiceProvider 內定義的元件別名
Blade::component('themes.default.components.modal', 'modal');

// 視圖內對 modal 元件的調用
@modal([...元件參數])
    @slot('modalContent')
        Modal body content
    @endslot
@endmodal
```

### 2. 元件工廠
主要透過定義工廠方法 create 來取得元件原始碼，第一參數為調用的元件類別名與方法之複合名稱，第二參數則為該元件方法所需參數。
```
// internaltide/componentInjection/src/Components/ComponentFactory.php

public function create($compoentClassMethod, ...$arguments){}
```
### 3. 套件設定檔
 - extra.namespace
   定義用戶端放置元件類別定義檔的目錄，若指定調用的類別不存在於該目錄，則返回套件目錄繼續尋找，若不存在則拋出例外。
 - modal.background
   定義套件內建 Modal 的背景，可以是圖檔 URL、顏色名稱(red)或16進位顏色代碼(#ffffff)
 - modal.vendor
   定義使用之 Modal 的類型，目前僅內建 Bootstrap Modal，選用後會需要 Boostrap的環境。
 - modal.headercolor
   定義 Modal Header 的字體顏色
 - modal.contentcolor
   定義 Modal Body Content 的字體顏色
```
// internaltide/componentInjection/config/component.php

return [
    'extra' => [
        'namespace' => 'App\Components',
    ],
    'modal' => [
        'background' => 'white',
        'vendor' => 'bootstrap',
        'headcolor' => 'text-primary',
        'contentcolor' => 'text-info'
    ]
];
```
### 4. Modal 元件類別
主要定義取得各種內建 Modal 外框方法的類別，目前以 Bootstrap Modal 為主。預計，下一個會是 Jquery UI Modal。
```
// ModalComponent.php

// 用以取得 Bootstrap Modal 外框，返回原始碼還會包含一些JS函數，如AJAX 取得資料寫入 Modal Body等...
public function frame($title='Undefined Title', $tpl=null)
{
    $vendor = config('component.modal.vendor');
    switch($vendor){
        case 'bootstrap':
            return $this->bootstrap($title, $tpl);
            break;
        case 'jqueryui':
            return $this->jqueryui($title, $tpl);
            break;
        default:
            return $this->bootstrap($title, $tpl);
            break;
    }
}

public function bootstrap($title, $tpl=null)
{
    return view('componentInjection::modalframe', [...]);
}

public function jqueryui($title, $tpl=null){}
```
### 5. Modal 視圖
#### Bootstrap Modal Frame
```
<div class="row">
    <div id="modal-dialog" class="modal fade" tabindex="-1" role="dialog" aria-hidden="true">
        <div class="modal-dialog" data-bg="{{ $modalBackground }}">
            <div class="modal-content">
                <!-- Modal Header -->
                <div class="modal-header {{ $headerColor }}">
                    <button type="button" class="close" data-dismiss="modal">
                        <span aria-hidden="true">×</span>
                    </button>
                    <h4 class="modal-title" id="myModalLabel">{{ $modalLabel }}</h4>
                </div>

                <!-- Modal Content -->
                <div id="modal-body" class="{{ $contentColor }}">
                    <div class="container-fluid">
                        <!-- Load By Ajax -->
                    </div>
                </div>

                <!-- Modal Footer -->
                <div class="modal-footer">
                    <button type="button" class="btn btn-primary" id="modalSave">Save</button>
                    <button type="button" class="btn btn-primary" id="modalOk">Ok</button>
                    <button type="button" class="btn btn-primary" id="modalCancel">Cancel</button>
                </div>
            </div>
        </div>
    </div>
</div>
<script>
    一些實用的 Javascript Modal 函數
</script>
```
#### Modal Form
主要提供用戶繼承用，內含一些必要區塊，讓用戶更專注在定義表單內容
```
<form class="form-horizontal" id="modal-form" method="POST" action="{{ $actionUrl }}">
    <table id="form-main">
      <input type="hidden" id="modal-targets" name="target_id">
      <tbody>
        @section('modalForm')
          <tr>
              <td colspan="2">
                <label for="mailSubject"><span class="star">*</span>Subject</label>
                <input type="text" id="mailSubject" name="subject" required>
              </td>
          </tr>
          <tr>
              <td colspan="2">
                <label for="mailbody"><span class="star">*</span>Mail</label>
                <textarea id="mailbody" name="content" rows="10" required></textarea>
              </td>
          </tr>
          <button type="submit" class=""></button>
        @show
      </tbody>
    </table>
    <div class="modal-hidden">
      @yield('modalHidden')
    </div>
    {{ csrf_field() }}
</form>
@yield('additional')
```

### 6. 套件 Facade
創建一個繼承自 Illuminate\Support\Facades\Facade 且包含了 getFacadeAccessor 實作的類別即可為套件提供靜態調用的能力。(Facade 數量並未設限，視需要可依職責建立多個)
```
namespace Internatide\ComponentInjection\Components;

use Illuminate\Support\Facades\Facade;

class ComponentFacade extends Facade
{
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        // 返回的即是在 Service Provider 中註冊綁定的名稱
        return 'component.factory';
    }
}
```
**getFacadeAccessor** 是用來返回服務提供者內註冊綁定的名稱，為後續從容器解析物件實例的依據。所有 Facade 的靜態調用過程都會轉移到 __callStatic() 這個魔術方法處理，而該方法首先就是從容器取出底層實例，接著將動作委派給實例進行處理。

> ## 定義服務提供者內容
### 1. Boot Method
#### 註冊套件視圖
為了讓 Laravel 正常使用套件的內建視圖，必須經過註冊套件視圖目錄的動作
```
$this->loadViewsFrom(__DIR__.'/Views', 'componentInjection');

如果你還希望將視圖發佈到用戶端供其客製，可以再額外定義視圖的發佈
$this->publishes([
    __DIR__.'/Views' => resource_path('views/vendor/componentInjection'),
]);
```
#### 發佈設定檔
為了將設定檔發佈到應用程式以便使用者自行配置，需進行設定檔發佈的動作
```
$this->publishes([
    __DIR__.'/../config/component.php' => config_path('component.php')
], 'config');
```

現在，套件使用者可以透過指令 vendor:publish 將套件設定檔複製到指定位置，並像存取其它設定檔一樣使用。另外，透過指定了發佈方法的第二參數(表示群組標籤)，套件使用者可以依群組個別進行發佈。
```
// 僅發佈群組標籤為 config 的設定檔
php artisan vendor:publish --tag=config
```


### 2. Register Method
#### 設定檔合併
透過合併，套件設定檔內容將會是套件設定檔與指定發佈副本的合併結果，套件使用者可以只定義要覆寫的設定選項，未設定項目則使用套件設定檔的預設值。
```
$this->mergeConfigFrom(__DIR__.'/../config/component.php', 'component');
```

#### 註冊綁定
根據服務內對外公開的幾個主要物件進行綁定
```
$this->app->bind('component.factory', function () {
    return new ComponentFactory();
});
```

> ## Discoovery 配置
為了讓安裝套件的使用者可以自動註冊需要的服務提供者及 Facade，可以設定套件的 Discovery 讓使用者有更佳的安裝體驗。以下設定需放在套件的 composer.json 中。
```
"extra": {
    "laravel": {
        "providers": [
            "Internaltide\\ComponentInjection\\ComponentInjectionServiceProvider"
        ],
        "aliases": {
            "Component": "Internaltide\\ComponentInjection\\Components\\ComponentFacade"
        }
    }
},
```

> ## 上傳套件到 GitHub
### 1. 建立本地倉儲
```
cd Packages/packages/internaltide/componentInjection
git init
git add .
git commit -m "Initial Commit."
```

### 2. 建立 GitHub 倉儲
請進到 Github 的 Repository 清單頁面，右上角有一個綠色的 "New" 按鈕，點擊即可開始建立Github 儲存庫的流程，依指示完成並取得儲存庫網址，之後會用到。

### 3. Git Push
```
git remote add origin https://github.com/Internaltide/ComponentInjection.git
git push -u origin master
```

### 4. Git Tag
下版本標籤為必要動作，否則 composer 安裝時，會要求使用者設定 minimum-stability，反而不便。另外，設定的版本標籤也代表著釋出的穩定版本。開發期間，則可於用戶端指定安裝版本 dev-master ，即可安裝最新版本。
```
git tag -a 1.0.0 -m "First version"
git push --tags
```

> ## 提供套件透過 Composer 安裝的能力
為了可以使用 Composer 安裝剛剛建立的套件， 必須讓 Composer 知道套件的來源位置，<br/>
方法有二：

### 1. 手動增加套件對應的 Repository 位置
直接於用戶端應用的 composer.json 定義套件儲存庫位置。注意!! 存放在 Github 這類的版本控制儲存庫，必須將 type 設定為 "vcs"。
```
"repositories": [
    {
        "type": "vcs",
        "url": "https://github.com/Internaltide/ComponentInjection.git"
    }
],
```

注意!! 此方式可能需要在安裝時提供 Github 的授權 token，Composer 會提供索取 token 的連結，請按指示建立 token 後，再將 token 貼回即可順利完成安裝。
```
please create a GitHub OAuth token to go over the API rate limit
Head to https://github.com/settings/tokens/new?scopes=repo&description=Composer+on+e42acb586036+2019-01-27+0133
to retrieve a token. It will be stored in "/root/.composer/auth.json" for future use by Composer.
Token (hidden):
```

### 2. 上傳到 Packagist
Packagist 是 Composer 主要的套件儲存庫，因此也可選擇將套件上傳至 Packagist。如此一來，便可以直接安裝，省去使用者自行設定儲存庫的動作。<br/>
參考網址：<br/>
https://oomusou.io/laravel/laravel-package-hello-world/#%E4%B8%8A%E5%82%B3Packagist