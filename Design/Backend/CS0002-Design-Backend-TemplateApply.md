# Gentelella Package Case Study - Backend Template Use Gentelella

>&nbsp;<br/>
> 案例名稱：**CS0002-Design-Backend-TemplateApply**<br/>
> 應用類型：**TemplateApply**<br/>
> 使用套件：**Gentelella**<br/>
> 套件來源：https://github.com/puikinsh/gentelella<br/>
> 套件版本：1.4<br/>
> 套件管理工具：yarn<br/>
> 框架版本：Laravel 5.6<br/>
> 案例目標：管理後台套用現成的模板，加快開發速度<br/>
>&nbsp;<br/>

> ## Laravel Blade 模板繼承回顧
大多數的網頁應用程式在不同頁面都會保持著相同的佈局方式，因此可透過Blade提供的模板<br/>
繼承功能來簡化每一個內容視圖的內容。首先定義出Layout佈局視圖，往後的其它視圖就可以使<br/>
用 **@extend** 來繼承定義好的佈局視圖。<br/>

**範例佈局**
```
// layouts/app.blade.php
<html>
    <head>
        <title>App Name - @yield('title')</title>
    </head>
    <body>
        @section('sidebar')
            <p>This is the sidebar header.</p>
        @show

        <div class="container">
            @yield('content')
        </div>
    </body>
</html>
```
**繼承範例佈局的子頁面**
```
@extend('layouts.app'); // 此內容頁將繼承上面定義的那個佈局內容

@section('title')
    Page Title
@endsection

@section('content')
    <p>Main Content</p>
@endsection

@section('sidebar')
    @parent
    <p>Sidebar Menu</p>
@endsection
```

**@yield** 指令是被用來顯示給定區塊的內容，所以 **@yield('content')** 就是用來顯示定義好的content區塊內容。<br/>
換言之，如果子頁面內有定義出content區塊的內容，那其內容就會注入到繼承來的Layout之中。

**@show** 與 **@parent** 兩個指令則被使用來與 **@section** 相互搭配，可以在Layout頁與繼承<br/>
Layout的子頁各自定義一部分內容，最後在渲染時將兩邊的內容合併輸出。<br/>

以上面為例，<br/>
範例佈局之sidebar區塊的@show用來直接產生子頁面的定義內容；<br/>
子頁面之sidebar區塊的@parent則會置換成範例佈局內所定義的內容。<br/>
最後子頁面渲染內容：
```
<html>
    <head>
        <title>App Name - Page Title</title>
    </head>
    <body>
        <p>This is the sidebar header.</p>  // 如果子頁面的sidebar區塊沒有@parent，則此行消失
        <p>Sidebar Menu</p>                 // 如果範例佈局的sidebar區塊沒有@show，則此行消失

        <div class="container">
            <p>Main Content</p>
        </div>
    </body>
</html>
```

> ## Package.json 套件版本控制回顧
專案發展到一定程度，不見得都能夠使用每一個套件的最新版本，因此套件版本管理<br/>
也是一項很重要的工作。以下是一些常用套件版本控制用的語法：<br/>
 - **1.2.1**
    - 直接指定並限制只能安裝的版本
 - **^1.2.1**
   - 可使用 >=1.2.1 但 <2.0.0 的版本，亦即只能使用相容版本
 - **latest**
   - 安裝當前發佈的最新版本，即直接使用yarn add指令安裝的預設行為
 - **^5.x**
   - 可使用 >=5.0.0 但 <6.0.0 的版本
 - **&gt;=3.0.0**
   - 可以使用3.0.0以上的所有版本。其它還有<、<=、=、>可以使用，亦可用AND、OR、||來合併使用
 - **1.30.2 - 2.30.2**
   - 可以使用1.30.2到2.30.2之間的所有版本
 - **git://github.com/user/project.git#commit-ish**
   - 直接使用Git Url來代表要安裝的依賴版本


> ## Gentelella 模板內容
### 模板Layout
 - Wrapper
```
{Head}
<body class="nav-md">
    <div class="container body">
        <div class="main_container">
            {Side Menu}
            {Top Menu}
            {Main Content}
            {Footer}
        </div>
    </div>
</body>
```
 - Top Menu
```
<div class="top_nav">
    <div class="nav_menu">
        <nav>
            {menu toggle icon}
            {quick func bar}
        </nav>
    </div>
</div>
```
 - Side Menu
```
<div class="col-md-3 left_col">
    <div class="left_col scroll-view">
        {menu profile quick info}
        {menu}
        {menu footer button}
    </div>
</div>
```
 - Main Content
```
<div class="right_col" role="main">
</div>
```
 - Footer
```
<footer>
</footer>
```

> ## Gentelella 模板套用過程
### 1. Laravel 前端環境調整
選用的模板屬於Bootstrap3的設計，但Laravel 5.6 的預設前端搭建工具為Bootstrap4 + Vue，<br/>
所以並不相容，經測試也確實會跑版。因此，在不變更Laravel框架版本的前題下，預設前端<br/>
搭建工具勢必要有所調整。

理論上，直接把Bootstrap套件降級後退一個大版本就好，但預設就是Bootstrap4環境，<br/>
Laravel是否為了Bootstrap4做了相關的整合設置呢？(想想，Laravel還提供了移除指令就知道了)<br/>
所以，為免意外，筆者還是選擇了一條較為崎嶇的路，先移除預設的搭建工具再重新配置。

#### I. 移除預設的搭建工具
執行Artisan指令移除預設的Bootstrap4與Vue，執行後會留下空白的SCSS文件與幾個通用的Javascript庫。
```
php artisan preset none
```
如果需要保留Vue，可以執行下面指令將Vue裝回去
```
php artisan preset vue
```
註：這裡的搭建工具指的即是官方文件的Scaffolding(鷹架)，Google翻譯跟一些陸方文件都是直接<br/>
        翻成**腳手架**。因為聽起著實彆扭且怪異，因此，本文中一律改用搭建工具來表示Scaffolding。<br/>

國外網站對於Scaffolding的相關翻譯如下：<br/>
**Scaffolding is a framework that allows you to do basic CRUD operations against your database with little or no code.**<br/>

**Scaffolding generally refers to a quickly set up skeleton for an app.**<br/>

**Just like a real scaffolding in a building construction site, scaffolding gives you some kind of a (fast, simplified, temporary) structure for your project**<br/>

> **[筆者斜斜念]**  所以咩!! 機器翻譯就算了，自己翻還照抄不修飾，腳手架是什麼鬼啦!!

#### II. Layout頁面調整
Gentelella預設提供兩種Layout，主要差異為固定側邊選單或固定Footer。<br/>
請隨意挑選一個做為專案起始Layout，再進行調整與修飾後上傳。<br/>

另外，有一點要注意的是，[套件說明頁](https://puikinsh.github.io/gentelella/)裡的Head設置有包含用來讓低版本IE可支援<br/>
responsive及Html5新特性的腳本，但實際的套件檔案本身其實已經移除該腳本的掛載。<br/>
因此若你想繼續提供低版本IE的支援，可以重新掛上該腳本，只是記得要邊更新一下<br/>
連結來源，因為說明頁上的腳本連結已經無效。
```
<!-- HTML5 shim and Respond.js for IE8 support of HTML5 elements and media queries -->
<!--[if lt IE 9]>
    <script src="https://oss.maxcdn.com/html5shiv/3.7.3/html5shiv.min.js"></script>
    <script src="https://oss.maxcdn.com/respond/1.4.2/respond.min.js"></script>
<![endif]-->
```

#### III. 安裝套件調整
**後端模板套件Gentelella**<br/>
透過webpack配置直接複製所需資源進行自動化編譯打包。部分通用型資源則<br/>
改由yarn套件來提供，Gentelella套件只取其專屬客製的資源。<br/>
詳情可參考後續的webpack配置內容。
```
"gentelella": "^1.4.0",
```

**Bootstrap**<br/>
預設的bootstrap3套件是用less，若要維持sass編譯，需額外安裝bootstrap-sass套件。<br/>
否則須將bootstrap另外獨立出來使用less API來編譯。這裡維持使用sass，所以安裝了<br/>
bootstrap-sass這個套件。
```
"bootstrap": "^3.3.7",
"bootstrap-sass": "^3.3.7",
```

**Jquery**<br/>
移除預設搭建工具會連同jquery一併移除，所以手動再加回去。<br/>
這裡是直接用最新版本3.0系列，套用Layout時並未發生問題，<br/>
只是模板用的是大版本2.0系列，若後續套版有發生問題可能需要降轉。
```
"jquery": "^3.2",
```

**Font-awesome**<br/>
模板需要，額外增加之套件。
```
"font-awesome": "^4.7.0",
```

**popper.js**<br/>
Bootstrap4會用到的依賴套件，主要提供支援Tooltip功能。移除搭建工具時，套件從package.json<br/>
中被移除，但app.js內卻仍有載入popper.js的行為，若直接運行yarn run dev就會發生編譯失敗。<br/>
觀察webpack配置文件，依然保留app.js的編譯行為，所以也不像是預期開發者移除預設搭建工具後<br/>
不會再編譯app.js文件。最後，為了避免編譯失敗，還是將popper.js裝了回去。
```
"popper.js": "^1.12"
```

**開始安裝套件**<br/>
將上述套件一一加入到package.json後，即可直接下指令重新安裝。<br/>
注意!! 請將套件全數放在devDependencies區塊，因為專案上線後<br/>
會直接使用打包後的文件，沒有必要在線上環境重新安裝一次原生套件。<br/>
```
yarn install
```

若您是使用yarn add的方式來逐一安裝套件，記得必須加上 **--dev** 這個選項，才能正確將套件安裝在devDependencies。
```
yarn add font-awesome --dev
```

**注意!!**
移除預設前端搭建工具後，整個node_modules目錄都會被砍掉，所以即使沒有異動package.json<br/>
內容，也務必重新執行yarn install進行套件安裝。另外，若是用Laravel Artisan指令重新安裝<br/>
其他搭建工具，框架也只是更新一下package.json，但不會幫你安裝。<br/>

使用Artisan安裝前端搭建工具後出現的提示訊息：<br/>
**please run "npm install && npm run dev" to compile your fresh scaffolding**

#### IV. 資源編譯配置調整
**前題說明**<br/>
在相關編譯配置內容開始前，先說明一下整體的配置方式。<br/>
典型Laravel提供的是將資源載入設定統一都放在resources/js/app.js跟resources/css/app.css。<br/>
然後在JS部分則提供extract API來將第三方通用腳本萃取出來放到指定對外的資源目錄。<br/>
一般的結果如下<br/>
 - app.js         異動頻繁的客製JS
 - vendor.js    第三方通用的JS
 - app.css

這裡的作法會是，app.js跟app.css統一只放第三方資源或一些跨主題通用資源。<br/>
剩餘異動頻繁的客製部分，則依主題各自打包成獨立的文件。<br/>
(**所以啦!! 這裡的設計手法只是將Gentelella模板當作其中一個主題樣式而已唷**)<br/>
結果可能會類似於<br/>
 - app.js                       第三方通用的JS
 - theme01/theme.js    主題一的客製JS
 - theme02/theme.js    主題二的客製JS
 - app.css                     第三方通用的CSS
 - theme01/theme.css  主題一的客製CSS
 - theme02/theme.css  主題二的客製CSS

**配置說明**<br/>
**app.scss**<br/>
下了preset none的指令後，app.scss的內容都會被清空，改由開發者自行配置。<br/>
這裡我們定義下面兩項配置，即Bootstrap跟Fontaweson兩個套件的scss文件載入。<br/>
(當然你也可以在webpack裡頭配置，將套件內編譯好的文件複製過來打包就好)
```
@import '~bootstrap-sass/assets/stylesheets/bootstrap';

@import '~font-awesome/scss/font-awesome';
```

**app.js**<br/>
這邊不動，維持既有的配置，客製部分不會放這邊。<br/><br/>

**webpack.min.js**<br/>
裡頭的配置，將會分成全域區塊與多個主題區塊。<br/>

<全域配置區塊><br/>
**a.** Js與Css資源編譯<br/>
注意，在使用bootstrap3的環境下，不能使用extract指令，會出現異常的行為。<br/>
雖然編譯過程並沒出錯，但最後打包的文件就是無法成功將jquery套件載入。<br/>
Laravel 5.6 特有的問題？其 extract API 與 Bootstrap**4** 有嚴重耦合？
```
mix.js('resources/assets/js/app.js', 'public/js').sass('resources/assets/sass/app.scss', 'public/css');
```

**b.** Font-awesome字形檔複製<br/>
這個字形檔非常重要，是Fontawesome必要檔案。
```
mix.copyDirectory('node_modules/font-awesome/fonts', 'public/fonts');
```

<預設主題配置區塊><br/>
**a.** 模板預設圖檔<br/>
因只用到部分預設圖，所以並沒有全部打包，可視需求變動
```
mix.copy('node_modules/gentelella/production/images/img.jpg','public/default/images/img.jpg');
```

**b.** 主題專用第三方資源<br/>
以Gentelella模板專用的animate.css為例，雖亦屬第三方資源，但並不認定其為通用。<br/>
因為我們可能會預期fontawesome是每個主題模板都會用到，但卻不會認為animate.css<br/>
會被每個主題模板來使用。<br/>

當然最後，這個資源依舊會打包進app.css中，但我們不會一開始就配置到resources/css/app.css<br/>
裡頭，而是延後的webpack的主題配置區塊來處理相關行為。
```
mix.styles([
    'public/css/app.css',
    'node_modules/gentelella/vendors/animate.css/animate.css',
], 'public/css/app.css');
```

**c.** 主題客製Js與Css
```
mix.styles([
    'node_modules/gentelella/build/css/custom.css',
], 'public/default/css/theme.css');

mix.scripts([
    'node_modules/gentelella/build/js/custom.js',
], 'public/default/js/theme.js');
```

<其它主題配置區塊><br/>
......

#### V. 執行工作流程作業
```
yarn run dev
```

### 2. Layout頁面配置
除了基本的Blade指令來控制資料傳遞進視圖以外，主要是配置一些必要的實作區塊。<br/>
如此一來，所有繼承該Layout的子頁面就可以透過定義同名的區塊來達到各頁面的差異性。<br/>
最基本一定含有主內容區塊，這裡來設計出額外三個區塊供子頁面定義自己專屬的CSS、<br/>
JS以及覆寫CSS。<br/>

 - **@yield('pagecss')**：置放於Html head tag底部，可掛載頁面專屬的CSS定義。
 - **@yield('overrideCss')**：置放於頁面主內容頂端，可置放一些覆寫原有定義的CSS。
 - **@yield('content')**： 各頁面的主內容區塊。
 - **@yield('pagejs')**：置放於Html body tag底部，可掛載頁面專屬的Javascript腳本。

### 3. 建立可配置選單
Pendding!!