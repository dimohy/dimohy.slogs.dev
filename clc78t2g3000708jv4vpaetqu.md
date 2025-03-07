---
title: "[WinUI] ToggleSwitch에 Command 속성 추가"
datePublished: Wed Dec 28 2022 05:54:50 GMT+0000 (Coordinated Universal Time)
cuid: clc78t2g3000708jv4vpaetqu
slug: winui-toggleswitch-command
tags: dotnet, winui3

---

UWP 및 WinUI 3의 \[ToggleSwitch\]([https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.toggleswitch?view=winrt-22621](https://learn.microsoft.com/en-us/uwp/api/windows.ui.xaml.controls.toggleswitch?view=winrt-22621)) 컨트롤은 특이하게도 `Command` 속성이 없습니다. MVVM에서 사용하기 위해서는 `CanExecute()` 에 따라 컨트롤을 활성화/비활성화 할 필요가 있습니다.

`Microsoft.Xaml.Interactivity.Behavior`를 이용해서 다음 처럼 사용하고 싶은데요

```xml
<ToggleSwitch IsOn="{x:Bind SelectedAttribute.IsKey, Mode=OneWay}">
    <interactivity:Interaction.Behaviors>
        <extensions:ToggleSwitchCommandBehavior Command="{x:Bind TogglePKCommand}" />
    </interactivity:Interaction.Behaviors>
</ToggleSwitch>
```

`ToggleSwitchCommandBehavior` 는 다음처럼 구현할 수 있습니다.

```csharp
using System.Windows.Input;

using Microsoft.UI.Xaml;
using Microsoft.UI.Xaml.Controls;
using Microsoft.UI.Xaml.Input;
using Microsoft.Xaml.Interactivity;

using Windows.System;

namespace WinUI.Extensions;

public class ToggleSwitchCommandBehavior : Behavior<ToggleSwitch>
{
    public readonly static DependencyProperty CommandProperty =
        DependencyProperty.Register(nameof(Command), typeof(ICommand), typeof(ToggleSwitchCommandBehavior),
            new PropertyMetadata(null, OnIsExternalChanged));

    public readonly static DependencyProperty CommandParameterProperty =
        DependencyProperty.Register(nameof(CommandParameter), typeof(object), typeof(ToggleSwitchCommandBehavior),
            new PropertyMetadata(null));

    private bool _isAccessUserInputProperty;

    public ICommand? Command
    {
        get => (ICommand)GetValue(CommandProperty);
        set => SetValue(CommandProperty, value);
    }

    public object? CommandParameter
    {
        get => GetValue(CommandParameterProperty);
        set => SetValue(CommandParameterProperty, value);
    }


    protected override void OnAttached()
    {
        AssociatedObject.PointerReleased += ToggleSwitch_PointerReleased;
        AssociatedObject.PreviewKeyDown += ToggleSwitch_PreviewKeyDown;
        AssociatedObject.Toggled += ToggleSwitch_Toggled;

        if (Command is not null)
        {
            Command_CanExecuteChanged(this, EventArgs.Empty);
        }
    }

    protected override void OnDetaching()
    {
        AssociatedObject.PointerReleased -= ToggleSwitch_PointerReleased;
        AssociatedObject.PreviewKeyDown -= ToggleSwitch_PreviewKeyDown;
        AssociatedObject.Toggled -= ToggleSwitch_Toggled;
    }

    private static void OnIsExternalChanged(object sender, DependencyPropertyChangedEventArgs args)
    {
        if (sender is not ToggleSwitchCommandBehavior @this)
            return;

        if (args.OldValue is ICommand oldCommand)
            oldCommand.CanExecuteChanged -= @this.Command_CanExecuteChanged;

        if (args.NewValue is ICommand newCommand)
        {
            newCommand.CanExecuteChanged += @this.Command_CanExecuteChanged;
            if (@this.AssociatedObject is not null)
                @this.Command_CanExecuteChanged(@this, EventArgs.Empty);
        }
    }

    private void Command_CanExecuteChanged(object? sender, EventArgs e)
    {
        if (Command is null)
            return;

        AssociatedObject.IsEnabled = Command.CanExecute(CommandParameter);
    }

    private void ToggleSwitch_PreviewKeyDown(object sender, KeyRoutedEventArgs e)
    {
        if (e.Key is VirtualKey.Space)
        {
            _isAccessUserInputProperty = true;
        }
    }

    private void ToggleSwitch_PointerReleased(object sender, PointerRoutedEventArgs e)
    {
        _isAccessUserInputProperty = true;
    }

    private void ToggleSwitch_Toggled(object sender, RoutedEventArgs e)
    {
        var toggleSwitch = (ToggleSwitch)sender;

        // 사용자 입력일 때만 처리하도록 한다.
        if (_isAccessUserInputProperty is false)
            return;
        _isAccessUserInputProperty = false;

        if (Command is null)
            return;

        // 커멘드에 의해 값이 변하도록 IsOn 값을 원복한다.
        toggleSwitch.IsOn = !toggleSwitch.IsOn;

        if (Command.CanExecute(CommandParameter))
            Command.Execute(CommandParameter);
    }
}
```

`ToggleSwitch`가 토글 되었을 때 사용자 입력에 위해서인지를 확인하기 위해서는 마우스 입력이나 키보드 입력을 감지해야 합니다.  
특이한 점은 WinUI의 `ToggleSwitch`는 `Tapped`이벤트가 발생하지 않는 것인데요, 다행히 `PointerReleased`이벤트는 발생하여 이것을 이용하였습니다.