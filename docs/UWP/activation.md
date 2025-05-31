# ActivationService & ActivationHandlers

:heavy_exclamation_mark: There is also a version of [this document with code in VB.Net](./activation.vb.md) :heavy_exclamation_mark: |
-------------------------------------------------------------------------------------------------------------------------------------------- |

## ActivationService

The ActivationService is in charge of handling the application's initialization and activation.

The method `ActivateAsync()` it the common entry point for the application lifecycle events `OnLaunched`, `OnActivated` and `OnBackgroundActivated`.
For more information on the application lifecycle and its events see [Windows 10 universal Windows platform (UWP) app lifecycle](https://docs.microsoft.com/windows/uwp/launch-resume/app-lifecycle).

## ActivationHandlers

The `ActivationService` relies on the ActivationHandlers, that are registered in the method `GetActivationHandlers()`.

Each class in the application that can handle activation derives from the abstract class `ActivationHandler<T>` (T is the type of ActivationEventArguments the class can handle) and implement the method HandleInternalAsync().
The method `HandleInternalAsync()` is where the actual activation takes place.
The virtual method `CanHandleInternal()` checks if the incoming activation arguments are of the type the ActivationHandler can manage. It can be overwritten to establish further conditions based on the ActivationEventArguments.

### ActivationHandlers sample

We'll have look at the SchemeActivationHandler, added by the [DeepLink](./features/deep-linking.md) feature, to see how activation works in detail:

```csharp
protected override bool CanHandleInternal(ProtocolActivatedEventArgs args)
{
    // If your app has multiple handlers of ProtocolActivationEventArgs
    // use this method to determine which to use. (possibly checking args.Uri.Scheme)
    return true;
}

// By default, this handler expects URIs of the format 'wtsapp:sample?paramName1=paramValue1&paramName2=paramValue2'
protected override async Task HandleInternalAsync(ProtocolActivatedEventArgs args)
{
    // Create data from activation Uri in ProtocolActivatedEventArgs
    var data = new SchemeActivationData(args.Uri);
    if (data.IsValid)
    {
        NavigationService.Navigate(data.PageType, data.Parameters);
    }

    await Task.CompletedTask;
}
```

The `CanHandleInternal()` method was overwritten here and it returns true by default. You can use `args` to add extra validation for scenarios with multiple `ProtocolActivationEventArgs`.

The `HandleInternalAsync()` method gets the `ActivationData` from argument's Uri and uses the `PageType` and `Parameters` to navigate.

## Activation in depth

### Activation flow

The following flowchart shows the activation process that starts with one of the app lifecycle event and ends with the `StartupAsync` call.

### Lifecycle event  `OnLaunched`, `OnActivated` or `OnBackgroundActivated`

Activation starts from one of the app lifecycle events: `OnLaunched`, `OnActivated` or `OnBackgroundActivated`. All call a common entry point for activation in `ActivationService.ActivateAsync()`.

![app lifecycle steps](./resources/activation/AppLifecycleEvent.png)

### ActivateAsync

The first calls in `ActivateAsync` are `InitializeAsync()` and ShellCreation (in the cases where activation is interactive). If you added an Identity feature to your app, code for Identity configuration and SilentLogin will be included here too.

After this first block, `HandleActivationAsync` is called (more details below).

![activation flow](./resources/activation/ActivateAsync.png)

**IsInteractive**

Interacting with the app window and navigating is only available when the activation arguments extend from `IActivatedEventArgs`. An example for non-interactive activation is activation from a background Task.

**InitializeAsync**

`InitializeAsync` contains services initialization for services that are going to be used as `ActivationHandler`. This method is called before the window is activated. Only code that needs to be executed before app activation should be placed here, as the splash screen is shown while this code is executed.

**StartupAsync**

`StartupAsync` contains initializations of other classes that do not need to happen before app activation and starts processes that will be run after the Window is activated.

### HandleActivation

The `HandleActivation` method gets the first `ActivationHandler` that can handle the arguments of the current activation. It also creates a `DefaultActivationHandler` and executes it if no other `ActivationHandler` is selected or the selected `ActivationHandler` does not result in Navigation.

![flow for handling activation](./resources/activation/HandleActivation.png)

## Sample: Add activation from File Association

We are going to create a new `ActivationHandler` to show how to extend the `ActivationService` in your project. In this case, we are going to add a reader for markdown (.md) files.

The following sample code is to be added in a *TS* generated app using the MVVM Toolkit.

For viewing the markdown a `MarkdownTextBlock` from the [Windows Community Toolkit](https://github.com/Microsoft/WindowsCommunityToolkit) is used.

### 1. Add Page and ViewModel to show the opened file

Add the following files to your project

**Views/MarkdownPage.xaml**

```xml
<Page
    x:Class="YourAppName.Views.MarkdownPage"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:controls="using:Microsoft.Toolkit.Uwp.UI.Controls"
    mc:Ignorable="d">
    <Grid
        x:Name="ContentArea">

        <Grid.RowDefinitions>
            <RowDefinition x:Name="TitleRow" Height="48"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>

        <TextBlock
            x:Name="TitlePage"
            x:Uid="Markdown_Title"
            FontSize="28" FontWeight="SemiLight"
            TextTrimming="CharacterEllipsis"
            TextWrapping="NoWrap" VerticalAlignment="Center"
            Margin="24,0,24,7"/>

        <Grid Grid.Row="1" >
            <ScrollViewer
                Margin="12"
                BorderBrush="{ThemeResource AppBarBorderThemeBrush}"
                BorderThickness="2"
                HorizontalScrollBarVisibility="Disabled"
                VerticalScrollBarVisibility="Visible">
                <controls:MarkdownTextBlock
                    x:Name="UiMarkdownText"
                    Padding="24,12,24,12"
                    Foreground="Black"
                    Background="White"
                    LinkClicked="UiMarkdownText_LinkClicked"
                    Text="{x:Bind ViewModel.Text, Mode=TwoWay}" />
            </ScrollViewer>
        </Grid>
    </Grid>
</Page>
```

**Views/MarkdownPage.xaml.cs**

```csharp
using System;
using Microsoft.Toolkit.Uwp.UI.Controls;
using Windows.Storage;
using Windows.System;
using Windows.UI.Xaml.Controls;
using Windows.UI.Xaml.Navigation;

using YourAppName.ViewModels;

namespace YourAppName.Views
{
    public sealed partial class MarkdownPage : Page
    {
        public MarkdownViewModel ViewModel { get; } = new MarkdownViewModel();

        public MarkdownPage()
        {
            InitializeComponent();
        }

        protected override async void OnNavigatedTo(NavigationEventArgs e)
        {
            if (!string.IsNullOrEmpty(e.Parameter?.ToString()))
            {
                var text = await FileIO.ReadTextAsync(e.Parameter as StorageFile);
                ViewModel.Text = text;
            }
        }

        private async void UiMarkdownText_LinkClicked(object sender, LinkClickedEventArgs e)
        {
            await Launcher.LaunchUriAsync(new Uri(e.Link));
        }
    }
}
```

**ViewModels/MarkdownViewModel.cs**

```csharp
using System;
using YourAppName.Helpers;

namespace YourAppName.ViewModels
{
    public class MarkdownViewModel : ObservableObject
    {
        private string _text;

        public string Text
        {
            get { return _text; }
            set { SetProperty(ref _text, value); }
        }

        public MarkdownViewModel()
        {
        }
    }
}
```

**Strings/en-us/Resources.resw**

You also should add a string resource for Markdown Title.

```xml
<data name="Markdown_Title.Text" xml:space="preserve">
    <value>Markdown</value>
</data>
```

### 2. Set up File Association Activation

Add a file type association declaration in the application manifest, to allow the App to be shown as a default handler for markdown files.

![screenshot of package.appxmanifest editor showing file type extension being set](./resources/activation/DeclarationFileAssociation.PNG)

Handle the file activation event by implementing the override of OnFileActivated:

**App.xaml.cs**

```csharp
protected override async void OnFileActivated(FileActivatedEventArgs args)
{
    await ActivationService.ActivateAsync(args);
}
```

### 3. Add a FileAssociationService

Add a `FileAssociationService` to your project that handles activation from files. It derives from `ApplicationHandler<T>`.
As it manages activation by `File​Activated​Event​Args` the signature is:

**FileAssociationService**

```csharp
internal class FileAssociationService : ActivationHandler<File​Activated​Event​Args>
{

}
```

Override `HandleInternalAsync()`, to evaluate the event args, and take action:

```csharp
protected override async Task HandleInternalAsync(File​Activated​Event​Args args)
{
    var file = args.Files.FirstOrDefault();

    NavigationService.Navigate(typeof(MarkdownPage), file);

    await Task.CompletedTask;
}
```

### 4. Add the FileAssociationService to ActivationService

Finally, we'll add our new `FileAssociationService` to the `ActivationHandlers` registered in the `ActivationService`:

**ActivationService**

```csharp
private IEnumerable<ActivationHandler> GetActivationHandlers()
{
    // Add this new FileAssociationService
    yield return Singleton<FileAssociationService>.Instance;
}
```

---

## Learn more

- [Using and extending the generated app](./getting-started-endusers.md)
- [Handling navigation within the app](./navigation.md)
- [Adapt the app for specific platforms](./platform-specific-recommendations.md)
- [All docs](../README.md)
