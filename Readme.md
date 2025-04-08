# Dynamic Loading Sample for Blazor WASM

<p align="center">
    <a href="Readmd.ja">🇯🇵 日本語</a> &nbsp;|&nbsp;
    <a href="Readme.md">🇺🇸 English</a>
</p>
---

This repository provides a sample for dynamic loading in Blazor WASM. It is based on [Lazy load assemblies in ASP.NET Core Blazor WebAssembly](https://learn.microsoft.com/en-us/aspnet/core/blazor/webassembly-lazy-load-assemblies?view=aspnetcore-9.0) and demonstrates a working example of dynamic loading in Blazor WASM.

## How to Run

1. Navigate to the DynamicLoad project:

```bash
cd DynamicLoad
```

2. Start the server:

```bash
dotnet run
```

3. Open in your browser:

```bash
http://localhost:5244
```

## Behavior

### Before LazyLoad
![Before LazyLoad](/docs/images/BeforeLazyLoad.png)

When you click on Home or other links, the default page is displayed first. The Blazor template is shown initially (Home, Counter, Weather). At this point, the dynamically loaded assemblies have not been loaded yet.

### After LazyLoad
![After LazyLoad](/docs/images/AfterLazyLoad.png)

When you click on the Lazy Component and navigate to `/robot`, LazyLoad is triggered. The corresponding component's `.wasm` and debug `.pdb` files are loaded.

## Configuration and Mechanism

The main project is `DynamicLoad`. This project includes a class library named `GrantImaharaRobotControls` as described in the documentation.

```bash
dotnet new razorclasslib -o GrantImaharaRobotControls
```

In the `GrantImaharaRobotControls` project, the following files are created:

- `HeadGesture.cs`
- `Robot.razor`

At the top of `Robot.razor`, the route component is specified:

```csharp
@page "/robot"
```

When navigating to `/robot`, `Robot.razor` is prepared for display. Add a reference to the main project:

DynamicLoad.csproj:

```xml
<ItemGroup>
        <ProjectReference Include="..\GrantImaharaRobotControls\GrantImaharaRobotControls.csproj" />
        <BlazorWebAssemblyLazyLoad Include="GrantImaharaRobotControls.dll" />
</ItemGroup>
```

Add the project reference and specify `BlazorWebAssemblyLazyLoad` for the assembly to be lazy-loaded. If there are dependencies, add each one to `BlazorWebAssemblyLazyLoad`.

In `App.razor`, configure the routes as follows:

DynamicLoad/App.razor:

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
```

Use DI to obtain `AssemblyLoader`. Add a conditional branch to lazy-load assemblies when navigating to `/robot`. The `OnNavigateAsync` method is called during page navigation. Use `AssemblyLoader.LoadAssembliesAsync` to specify the assemblies to lazy-load. In this example, `GrantImaharaRobotControls.dll` is specified. Add it to `lazyLoadedAssemblies` and use the `grantImaharaRobotControlsAssemblyLoaded` flag to check if it has already been loaded.

With this setup, you can lazy-load assemblies when navigating to specific pages.

### Optional: DI Configuration in Program.cs

If you want to configure DI in `Program.cs`, do the following:

```csharp
// Lazy Load configuration
builder.Services.AddTransient<LazyAssemblyLoader>();

// Example usage: Dynamically load assemblies using LazyAssemblyLoader
var lazyAssemblyLoader = builder.Services.BuildServiceProvider().GetRequiredService<LazyAssemblyLoader>();

// Load required assemblies
await lazyAssemblyLoader.LoadAssemblyAsync("DynamicLoad.SomeLazyLoadedAssembly.dll");
```

## Notes

- Specify the assemblies to lazy-load according to your use case.
- Assemblies to be lazy-loaded must be specified with `BlazorWebAssemblyLazyLoad`.
- Consider whether splitting DLLs is necessary from a performance perspective.

## References

- [Lazy load assemblies in ASP.NET Core Blazor WebAssembly](https://learn.microsoft.com/en-us/aspnet/core/blazor/webassembly-lazy-load-assemblies?view=aspnetcore-9.0)