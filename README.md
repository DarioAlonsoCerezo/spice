# Spice 🌶, a spicy cross-platform UI framework!

A prototype (and design) of API minimalism for mobile.

If you like this idea, star for approval! Read on for details!

## Background & Motivation

In reviewing, many of the *cool* UI frameworks for mobile:

* [Flutter](https://flutter.dev)
* [SwiftUI](https://developer.apple.com/xcode/swiftui/)
* [Jetpack Compose](https://developer.android.com/jetpack/compose)
* [Fabulous](https://fabulous.dev/)
* [Comet](https://github.com/dotnet/Comet)
* An, of course, [.NET MAUI](https://dotnet.microsoft.com/apps/maui)!

Looking at what apps look like today -- it seems like bunch of
rigamarole to me. Can we build mobile applications *without* design
patterns?

The idea is we could build apps in a simple way, in a similar vein as
[minimal APIs in ASP.NET Core][minimal-apis] but for mobile & maybe
one day desktop:

```csharp
public class App : Application
{
    public App()
    {
        int count = 0;
    
        var label = new Label
        {
            Text = "Hello, Spice! 🌶",
        };
    
        var button = new Button
        {
            Text = "Click Me",
            Clicked = _ => label.Text = $"Times: {++count}"
        };
    
        Main = new StackView { label, button };
    }
}
```

These "view" types are mostly just [POCOs][poco].

Thus you can easily write unit tests in a vanilla `net7.0` Xunit
project, such as:

```csharp
[Fact]
public void Application()
{
    var app = new App();
    Assert.Equal(2, app.Main.Children.Count);

    var label = app.Main.Children[0] as Label;
    Assert.NotNull(label);
    var button = app.Main.Children[1] as Button;
    Assert.NotNull(button);

    button.Clicked(button);
    Assert.Equal("Times: 1", label.Text);

    button.Clicked(button);
    Assert.Equal("Times: 2", label.Text);
}
```

The above views in a `net7.0` project are not real UI, while
`net7.0-android` and `net7.0-ios` projects get the full
implementations that actually *do* something on screen.

So for example, adding `App` to the screen on Android:

```csharp
protected override void OnCreate(Bundle savedInstanceState)
{
    base.OnCreate(savedInstanceState);

    SetContentView(new App());
}
```

And on iOS:

```csharp
var vc = new UIViewController();
vc.View!.AddSubview(new App());
Window.RootViewController = vc;
```

`App` is a native view on both platforms. You just add it to an
existing app as you would any other control or view.

[poco]: https://en.wikipedia.org/wiki/Plain_old_CLR_object
[minimal-apis]: https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis

## Scope

* No XAML. No DI. No MVVM. No MVC. No data-binding. No System.Reflection.
* Target iOS & Android only to start.
* Implement only the simplest controls.
* The native platforms do their own layout.
* Document how to author custom controls.
* Leverage C# Hot Reload for fast development.
* Measure startup time & app size.
* Profit?

Benefits of this approach are full support for trimming and eventually
[NativeAOT][nativeaot] if it comes to mobile one day. 😉

[nativeaot]: https://learn.microsoft.com/dotnet/core/deploying/native-aot/

## Thoughts on .NET MAUI

.NET MAUI is great. XAML is great. Think of this idea as a "mini"
MAUI.

Spice will even leverage various parts of .NET MAUI:

* The iOS and Android workloads for .NET.
* The .NET MAUI "Single Project" system.
* The .NET MAUI "Asset" system, aka Resizetizer.
* Microsoft.Maui.Graphics for primitives like `Color`.

And, of course, you should be able to use Microsoft.Maui.Essentials by
opting in with `UseMauiEssentials=true`.

It is an achievement in itself that I was able to invent my own UI
framework and pick and choose the pieces of .NET MAUI that made sense
for my framework.

## Getting Started

Simply install the template:

```bash
dotnet new install Spice.Templates
```

Create the project and build it as you would for other .NET projects:

```bash
dotnet new spice
dotnet build
```

## Implemented Controls

* `View`: maps to `Android.Views.View` and `UIKit.View`.
* `Label`: maps to `Android.Widget.TextView` and `UIKit.UILabel`
* `Button`: maps to `Android.Widget.Button` and `UIKit.UIButton`
* `StackView`: maps to `Android.Widget.LinearLayout` and `UIKit.UIStackView`
* `Image`: maps to `Android.Widget.ImageView` and `UIKit.UIImageView`

## Custom Controls

Let's review an implementation for `Image`.

First, you can write the cross-platform part for a vanilla `net7.0`
class library:

```csharp
public partial class Image : View
{
    [ObservableProperty]
    string _source = "";
}
```

`[ObservableProperty]` comes from the [MVVM Community
Toolkit][observable] -- I made use of it for simplicity. It will
automatically generate various `partial` methods,
`INotifyPropertyChanged`, and a `public` property named `Source`.

We can implement the control on Android, such as:

```csharp
public partial class Image
{
    public static implicit operator ImageView(Image image) => image.NativeView;

    public Image() : base(c => new ImageView(c)) { }

    public new ImageView NativeView => (ImageView)_nativeView.Value;

    partial void OnSourceChanged(string value)
    {
        // NOTE: the real implementation is in Java for performance reasons
        var image = NativeView;
        var context = image.Context;
        int id = context!.Resources!.GetIdentifier(value, "drawable", context.PackageName);
        if (id != 0) 
        {
            image.SetImageResource(id);
        }
    }
}
```

This code takes the name of an image, and looks up a drawable with the
same name. This also leverages the .NET MAUI asset system, so a
`spice.svg` can simply be loaded via `new Image { Source = "spice" }`.

Lastly, the iOS implementation:

```csharp
public partial class Image
{
    public static implicit operator UIImageView(Image image) => image.NativeView;

    public Image() : base(_ => new UIImageView { AutoresizingMask = UIViewAutoresizing.None }) { }

    public new UIImageView NativeView => (UIImageView)_nativeView.Value;

    partial void OnSourceChanged(string value) => NativeView.Image = UIImage.FromFile($"{value}.png");
}
```

This implementation is a bit simpler, all we have to do is call
`UIImage.FromFile()` and make sure to append a `.png` file extension
that the MAUI asset system generates.

[observable]: https://learn.microsoft.com/dotnet/communitytoolkit/mvvm/generators/observableproperty

## Hot Reload

C# Hot Reload (in Visual Studio) works fine, as it does for vanilla .NET
iOS/Android apps:

![Hot Reload Demo](docs/hotreload.gif)

Note that this only works for `Button.Clicked` because the method is
invoked when you click. If the method that was changed was already
run, *something* has to force it to run again.
[`MetadataUpdateHandler`][muh] is the solution to this problem, giving
frameworks a way to "reload themselves" for Hot Reload.

Unfortunately, [`MetadataUpdateHandler`][muh] does not currently work
for non-MAUI apps in Visual Studio 2022 17.5:

```csharp
[assembly: System.Reflection.Metadata.MetadataUpdateHandler(typeof(HotReload))]

static class HotReload
{
    static void UpdateApplication(Type[]? updatedTypes)
    {
        if (updatedTypes == null)
            return;
        foreach (var type in updatedTypes)
        {
            // Do something with the type
            Console.WriteLine("UpdateApplication: " + type);
        }
    }
}
```

The above code works fine in a `dotnet new maui` app, but not a
`dotnet new spice` or `dotnet new android` application.

And so we can't add proper functionality for reloading `ctor`'s of
Spice views. The general idea is we could recreate the `App` class and
replace the views on screen. We could also create Android activities
or iOS view controllers if necessary.

Hopefully, we can implement this for a future release of Visual Studio.

[muh]: https://learn.microsoft.com/dotnet/api/system.reflection.metadata.metadataupdatehandlerattribute

## Startup Time & App Size

In comparison to a `dotnet new maui` project.

Startup time for a `Release` build on a Pixel 5:

Spice:
```log
02-02 17:02:49.152  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +272ms
02-02 17:02:50.578  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +241ms
02-02 17:02:52.003  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +251ms
02-02 17:02:53.413  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +261ms
02-02 17:02:54.825  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +254ms
02-02 17:02:56.249  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +260ms
02-02 17:02:57.673  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +266ms
02-02 17:02:59.112  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +258ms
02-02 17:03:00.531  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +249ms
02-02 17:03:01.952  2174  2505 I ActivityTaskManager: Displayed com.companyname.HeadToHeadSpice/crc6421a68941fd0c4613.MainActivity: +252ms
Average(ms): 256.4
Std Err(ms): 2.82528268005561
Std Dev(ms): 8.93432830280051
```

.NET MAUI:
```log
02-02 17:05:02.946  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +543ms
02-02 17:05:04.877  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +528ms
02-02 17:05:06.593  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +546ms
02-02 17:05:08.278  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +540ms
02-02 17:05:09.979  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +529ms
02-02 17:05:11.745  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +544ms
02-02 17:05:13.471  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +553ms
02-02 17:05:15.197  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +533ms
02-02 17:05:16.904  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +539ms
02-02 17:05:18.646  2174  2505 I ActivityTaskManager: Displayed com.companyname.headtoheadmaui/crc649f845fb8d5de61df.MainActivity: +567ms
Average(ms): 542.2
Std Err(ms): 3.69022733415948
Std Dev(ms): 11.6695234597552
```

App size:

```
 9222186 com.companyname.HeadToHeadSpice-Signed.apk
15180467 com.companyname.HeadToHeadMaui-Signed.apk
```

This gives you an idea of how much "stuff" is in .NET MAUI.
