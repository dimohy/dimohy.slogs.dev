## [WinUI 3] Creating WinUI 3 App that supports themes including titlebar

## Summary

WinUI 3 provides a theme system related to the theme of Windows. Let's see how to adjust the color of the titlebar to match the theme along with the theme function provided by WinUI 3. 

## Development Environment

- Visual Studio 2022 Preview 4
- .NET 6 RC1
- Windows App SDK Preview 2 

## Theme behavior in WinUI 3

| Windows theme color is light
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633921056583/-Lffmxxt9.png)

| Windows theme color is dark
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633921073298/Z_FWK76Te.png)

But the titlebar can't keep up with the theme!

## Theme settings in WinUI 3

Windows 10(11) supports light and dark themes. For Windows applications, you can apply a theme to this theme. This is possible through the following features: 

| FrameworkElement.cs
```csharp
public ElementTheme ActualTheme { get; }
public ElementTheme RequestedTheme { get; set; }
```

In WinUI 3, controls implemented through FrameworkElement can use the `ActualTheme` property to determine the currently selected theme. And you can set a theme in the `RequestedTheme` property. The available themes are as follows. 

| ElementTheme.cs
```csharp
    public enum ElementTheme
    {
        Default,
        Light,
        Dark
    }
```

Theme settings are typically applied to all controls under Content by setting the RequestedTheme property of Window.Content. To apply locally, this is possible by setting the RequestedTheme of the container to apply locally.

You can subscribe to events to act on when the theme changes. 

```csharp
rootElement.ActualThemeChanged += (s, e) => { };
```

This event can detect a theme change in the system as well as a user changing a theme. 

## Apply WinUI 3 titlebar theme

As you can see above, even if you set the window color theme to dark, the dark theme doesn't apply to the titlebar because the titlebar is not a component of WinUI 3. To improve this, set the theme type and window theme color change and change the color of the titlebar through the current `ActualTheme` property and the `ActualThemeChanged` event.

To change the title bar, you need the help of AppWindow, which you can get: 

```csharp
    public AppWindow AppWindow
    {
        get
        {
            var hWnd = WinRT.Interop.WindowNative.GetWindowHandle(this);
            var winId = Win32Interop.GetWindowIdFromWindow(hWnd);
            return AppWindow.GetFromWindowId(winId);
        }
    }
```

And the different colors of the title bar can be changed like this: 

```csharp
    private void ModifyTitlebarTheme()
    {
        var content = (Content as FrameworkElement)!;
        var value = content.ActualTheme;

        var titleBar = AppWindow.TitleBar;
        if (value == ElementTheme.Light)
        {
            titleBar.ForegroundColor = Colors.Black;
            titleBar.BackgroundColor = Colors.White;
            titleBar.InactiveForegroundColor = Colors.Gray;
            titleBar.InactiveBackgroundColor = Colors.White;

            titleBar.ButtonForegroundColor = Colors.Black;
            titleBar.ButtonBackgroundColor = Colors.White;
            titleBar.ButtonInactiveForegroundColor = Colors.Gray;
            titleBar.ButtonInactiveBackgroundColor = Colors.White;

            titleBar.ButtonHoverForegroundColor = Colors.Black;
            titleBar.ButtonHoverBackgroundColor = Color.FromArgb(255, 245, 245, 245);
            titleBar.ButtonPressedForegroundColor = Colors.Black;
            titleBar.ButtonPressedBackgroundColor = Colors.White;
        }
        else if (value == ElementTheme.Dark)
        {
            titleBar.ForegroundColor = Colors.White;
            titleBar.BackgroundColor = Color.FromArgb(255, 31, 31, 31);
            titleBar.InactiveForegroundColor = Colors.Gray;
            titleBar.InactiveBackgroundColor = Color.FromArgb(255, 31, 31, 31);

            titleBar.ButtonForegroundColor = Colors.White;
            titleBar.ButtonBackgroundColor = Color.FromArgb(255, 31, 31, 31);
            titleBar.ButtonInactiveForegroundColor = Colors.Gray;
            titleBar.ButtonInactiveBackgroundColor = Color.FromArgb(255, 31, 31, 31);

            titleBar.ButtonHoverForegroundColor = Colors.White;
            titleBar.ButtonHoverBackgroundColor = Color.FromArgb(255, 51, 51, 51);
            titleBar.ButtonPressedForegroundColor = Colors.White;
            titleBar.ButtonPressedBackgroundColor = Colors.Gray;
        }

        // Fixed bug where icon background color was not applied 
        titleBar.IconShowOptions = IconShowOptions.HideIconAndSystemMenu;
        titleBar.IconShowOptions = IconShowOptions.ShowIconAndSystemMenu;
    }
```

And subscribe to events to detect when the window's theme color changes. 

```csharp
    public MainWindow()
    {
        this.InitializeComponent();

        var content = (Content as FrameworkElement)!;
        content.ActualThemeChanged += (s, e) =>
        {
            ModifyTitlebarTheme();
        };
        ModifyTitlebarTheme();
    }
```

## Action Screen

| If the window theme color is set to light 
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633921538224/2Ne68Sj0u.png)

| If the window theme color is set to dark
![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1633921553457/Vr2C0iukA.png)
