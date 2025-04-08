# Blazor WASM の動的ロードサンプル

<p align="center">
  <a href="Readme.ja.md">🇯🇵 日本語</a> &nbsp;|&nbsp;
  <a href="Readme.md">🇺🇸 English</a>
</p>
---

このレポでは Blazor WASM の動的ロードのサンプルを提供します。
このレポでは[ASP.NET Core でのアセンブリの遅延読み込み Blazor WebAssembly](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/webassembly-lazy-load-assemblies?view=aspnetcore-9.0)をベースに実際に動いている Blazor WASM の動的ロードのサンプルを提供します。

## 動かし方

1. DynamicLoad Project に移動

```bash
cd DynamicLoad
```

2. サーバーを起動

```bash
dotnet run
```

3. ブラウザで表示

```bash
http://localhost:5244
```

## 動作確認

LazyLoad 前
![LazyLoad前](/docs/images/BeforeLazyLoad.png)

Home などをクリックするとまずはデフォルトのページが表示されます。
Blazor のテンプレートがまずは表示されます。(Home, Counter, Weather)

この時には動的にロードされるアセンブリはまだロードされていません。

LazyLoad 後
![LazyLoad後](/docs/images/AfterLazyLoad.png)

Lazy Component をクリックすると`/robot`に遷移すると、LazyLoad が実行されます。
該当のコンポーネントの wasm とデバック用の dbb がロードされています。

## 設定と仕組み

メインとなるプロジェクトは DynamicLoad です。
本プロジェクトはドキュメントに記載の GrantmaharaRobotContols をクラスライブラリとして作成しています。

```bash
dotnet new razorclasslib -o GrantImaharaRobotControls
```

そちらで作成された`GrantImaharaRobotControls`プロジェクトに以下のクラスと razor ファイルを作成しています。

- `HeadGesture.cs`
- `Robot.razor`

Robot.razor のトップにはルートコンポーネントを指定しています。

```charp
@page "/robot"
```

`/robot`に遷移すると、Robot.razor が表示される準備をします。

作成したらメインプロジェクトから参照を追加します。

DynamicLoad.csproj

```xml
<ItemGroup>
    <ProjectReference Include="..\GrantImaharaRobotControls\GrantImaharaRobotControls.csproj" />
    <BlazorWebAssemblyLazyLoad Include="GrantImaharaRobotControls.dll" />
</ItemGroup>
```

作成したプロジェクトのReferenceを追加し、さらに`BlazorWebAssemblyLazyLoad`を追加します。
`BlazorWebAssemblyLazyLoad`は、LazyLoad するアセンブリを指定します。

依存関係がある場合一つずつ`BlazorWebAssemblyLazyLoad`を追加していく必要があります。

その上でApp.razorには以下のようにルートを設定します。

`Dynamicload/App.razor`

```csharp
@using GrantImaharaRobotControls
@using Microsoft.AspNetCore.Components.WebAssembly.Services
@inject LazyAssemblyLoader AssemblyLoader

<Router AppAssembly="typeof(App).Assembly"
        AdditionalAssemblies="lazyLoadedAssemblies"
        OnNavigateAsync="OnNavigateAsync">
    <Navigating>
        <div style="padding:20px;background-color:blue;color:white">
            <p>Loading the requested page&hellip;</p>
        </div>
    </Navigating>
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(MainLayout)" />
    </Found>
    <NotFound>
        <LayoutView Layout="typeof(MainLayout)">
            <p>Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>

@code {
    private List<Assembly> lazyLoadedAssemblies = [];
    private bool grantImaharaRobotControlsAssemblyLoaded;

    private async Task OnNavigateAsync(NavigationContext args)
    {
        try
        {
            if ((args.Path == "robot") && !grantImaharaRobotControlsAssemblyLoaded)
            {
                var assemblies = await AssemblyLoader.LoadAssembliesAsync(
                    [ "GrantImaharaRobotControls.dll" ]);
                lazyLoadedAssemblies.AddRange(assemblies);
                grantImaharaRobotControlsAssemblyLoaded = true;
            }
        }
        catch (Exception ex)
        {
            Logger.LogError("Error: {Message}", ex.Message);
        }
    }
}
}
```

`AssemblyLoader` を DI で取得します。
`Path`が`/robot`に遷移しか追加のアセンブリが読まれていないた場合に、LazyLoad する条件分岐を追加します。
`OnNavigateAsync`メソッドは、ページ遷移時に呼び出されます。
`AssemblyLoader.LoadAssembliesAsync`メソッドで、LazyLoad するアセンブリを指定します。
この例では`GrantImaharaRobotControls.dll`を指定しています。
`lazyLoadedAssemblies`に追加していきます。
`grantImaharaRobotControlsAssemblyLoaded`フラグで、
`GrantImaharaRobotControls.dll`がロード済みかどうかを判定します。

上記のように設定することで特定のページに遷移した際に、LazyLoad することができます。

### オプション Program.csでのDIの設定

もし`Program.cs`で DI の設定を行う場合は以下のようにします。


```csharp
// Lazy Load設定
builder.Services.AddTransient<LazyAssemblyLoader>();

// 使用例: LazyAssemblyLoaderを利用してアセンブリを動的にロードする
var lazyAssemblyLoader = builder.Services.BuildServiceProvider().GetRequiredService<LazyAssemblyLoader>();

// 必要なアセンブリをロード
await lazyAssemblyLoader.LoadAssemblyAsync("DynamicLoad.SomeLazyLoadedAssembly.dll");
```

## 注意点

- ユースケースに合わせて、LazyLoad するアセンブリを指定する必要があります。
- LazyLoad するアセンブリは、`BlazorWebAssemblyLazyLoad`を指定する必要があります。
- パフォーマンスの観点を考えるとそもそもdllを分ける必要があるのか？という点も考慮する必要があります。

## 参考

- [ASP.NET Core でのアセンブリの遅延読み込み Blazor WebAssembly](https://learn.microsoft.com/ja-jp/aspnet/core/blazor/webassembly-lazy-load-assemblies?view=aspnetcore-9.0)