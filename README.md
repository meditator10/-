# 学习记录
这是一个加了注释版本的键鼠行为记录器，原仓库地址 https://github.com/guinhx/ActionRecorder

# 2、业务场景
这里所有的按钮逻辑实现都是通过事件触发。

1、记录键盘和鼠标的动作。（OnClickRecord【F9】） 

2、回放录制的动作。（OnClickPlayOrStop【F10】）

3、导出记录到文件，或者导入记录的文件到程序（OnClickImport    OnClickExport）

4、通过使用“循环回放”选项无限地复制它。（OnClickLoop）

5、使用“停止回放”选项，在没有按下任何键的情况下重现忽略所有鼠标移动动作（OnClickSuppressMouseMovePath）

6、使用倍增器或设定固定时间来控制回放的速度
（OnChangeSpeedType）（OnChangeSpeedMultiplier    OnChangeSpeedFixed）

# 3、View
这里 定义了两个样式：UnstyledButton和SideMenuItem。用于自定义WPF中Button和ListViewItem控件。  

1、UnstyledButton这个样式用于自定义Button控件，使其具有透明的背景和一个简单的模板。
Background属性被设置为Transparent，使按钮背景变为透明。
Template定义了按钮的外观。这里使用了一个简单的ControlTemplate，其中包含一个Grid，它的Background绑定到按钮的Background属性。ContentPresenter用于显示按钮的内容，并且内容对齐方式设置为左对齐和垂直居中。

2、 SideMenuItem这个样式用于自定义ListViewItem控件的外观和行为。

``` XML
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:fw="clr-namespace:SourceChord.FluentWPF;assembly=FluentWPF"
>
    <Style x:Key="UnstyledButton" TargetType="Button">
        <Setter Property="Background" Value="Transparent"/>
        <Setter Property="Template">
            <Setter.Value>
                <ControlTemplate TargetType="Button">
                    <Grid Background="{TemplateBinding Background}">
                        <ContentPresenter HorizontalAlignment="Left" VerticalAlignment="Center"/>
                    </Grid>
                </ControlTemplate>
            </Setter.Value>
        </Setter>
    </Style>

    <Style x:Key="SideMenuItem" TargetType="ListViewItem">
        <Setter Property="Padding" Value="0"/>
        <Setter Property="Focusable" Value="False"/>
        <Setter Property="BorderThickness" Value="0, 0, 0, 1"/>
        <Setter Property="BorderBrush" Value="#FF2B2B2B"/>
        <Setter Property="Foreground" Value="#FFC1C1C1"/>
    </Style>
</ResourceDictionary>
``` 
StackPanel是一种简单的布局容器，按照水平或垂直方向依次排列其子元素，而DockPanel是一种相对复杂的容器，可以将子元素停靠在容器的不同位置，可以嵌套使用<StackPanel DockPanel.Dock="Top">
``` XML
ListView
ListViewItem
TextBlock
TextBox
CheckBox
```
对于 MouseDown="Window_MouseDown" ,是窗口拖动：在没有标题栏的窗口中，通过响应MouseDown事件并调用DragMove()方法，可以实现窗口的拖动。 
``` C#
private void Window_MouseDown(object sender, MouseButtonEventArgs e)
{
    if (e.LeftButton == MouseButtonState.Pressed)
    {
        DragMove();
    }
}
```
# 4、ViewModel逻辑
● 单例模式：Application类通过静态属性Instance实现了单例模式，这意味着整个应用程序中只能有一个Application实例。这种模式通常用于确保某个类只有一个实例，并且提供一个全局访问点。

● 依赖注入：Application类在构造时接收了MainWindow的实例，并将其保存在内部字段中。这种设计可以用于管理应用程序的主窗口，并在需要时访问和操作它。

● 关联：MainWindow的构造函数中实例化了Application类，并将自身作为参数传递给它。这表明MainWindow和Application之间存在紧密的关联，MainWindow通过这种方式将自己注册到Application类中。

``` C#
public class Application
{        
    private readonly MainWindow _mainWindow;
    public static Application Instance { get; private set; }        

    public Application(MainWindow mainWindow)
    {
        Instance = this;
        _mainWindow = mainWindow;
    }
}
public partial class MainWindow
{
    public MainWindow()
    {
        InitializeComponent();
        _application = new Application(this);
    }
}
```
# 5、View和ViewModel同步更新
这里有一个日志循环打印任务， 通过异步编程方式，避免阻塞主线程，使得UI在日志渲染过程中仍然保持响应。

● 线程安全性：使用ConcurrentQueue确保日志队列在多线程环境下的安全访问。多个线程可以安全地将日志添加到队列中，UI线程则通过Dispatcher从队列中取出并显示日志。

● UI更新：由于WPF中的UI元素只能在创建它们的线程上被访问（通常是主线程）， 也就是只有UI线程可以直接更新UI元素，因此当需要从非UI线程更新UI时，必须通过Dispatcher进行操作。Invoke方法用于同步地在UI线程上执行指定的委托。 通过Dispatcher.Invoke确保了日志的追加操作在UI线程上进行，以避免跨线程操作UI引发的异常。

● 异步日志显示：使用DispatcherPriority.Background保证日志的显示不会阻塞UI线程的其他重要操作，提供了更好的用户体验。

``` C#
private readonly ConcurrentQueue<string> _logsRenderQueue = new ConcurrentQueue<string>();

_logger.Dispatcher.Invoke(() =>
                          {
                              var logs = string.Empty;
                              while (_logsRenderQueue.TryDequeue(out var log))
                                  logs += log;
                              if (logs.Length == 0)
                                  return;
                              _logger.AppendText(logs);
                              _logger.ScrollToEnd();
                          }, DispatcherPriority.Background);
```
