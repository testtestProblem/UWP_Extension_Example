# Why need UWP extention
Universal Windows Platform (UWP). It has delicate UI by XAML. But it can not support a lot of WIN32 function. That is why we need UWP extention to unlimited it ability. 

# Project describe
* Paltform: visual studio 2019  
* Function describe: UWP through Windows Desktop Extensions for the UWP tool side loading console.  
* Compile sample: UWP can not run at any CPU, and target should set Windows Application Packaging Project.  
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/b60b14bc-4c7e-4bba-9e72-c146665b3147)

# Just sideload other app

* This UWP_extention can side loading winform by using ```FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync()```.  
 
* Modify WapProj/Package.appxmanifest by adding extensions in Windows Application Packaging Project.
```XML
<Package
  xmlns="http://schemas.microsoft.com/appx/manifest/foundation/windows10"
  xmlns:uap="http://schemas.microsoft.com/appx/manifest/uap/windows10"
  xmlns:rescap="http://schemas.microsoft.com/appx/manifest/foundation/windows10/restrictedcapabilities"
  
  xmlns:mp="http://schemas.microsoft.com/appx/2014/phone/manifest"
  xmlns:desktop="http://schemas.microsoft.com/appx/manifest/desktop/windows10"
  IgnorableNamespaces="uap mp desktop rescap">
```

```XML
 <Applications>
    <Application Id="App"
      Executable="$targetnametoken$.exe"
      EntryPoint="$targetentrypoint$">
      <uap:VisualElements
        DisplayName="WapProj_HotTab_Win10"
        Description="WapProj_HotTab_Win10"
        BackgroundColor="transparent"
        Square150x150Logo="Images\Square150x150Logo.png"
        Square44x44Logo="Images\Square44x44Logo.png">
        <uap:DefaultTile Wide310x150Logo="Images\Wide310x150Logo.png" />
        <uap:SplashScreen Image="Images\SplashScreen.png" />
      </uap:VisualElements>
		
	 <Extensions>
		 <uap:Extension Category="windows.appService">
		 	<uap:AppService Name="SampleInteropService" />
		 </uap:Extension>
		 <desktop:Extension Category="windows.fullTrustProcess" Executable="CollectDataAP\CollectDataAP.exe" />
	 </Extensions>
		
    </Application>
  </Applications>
```

In WapProj/Dependencies 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/8e778761-9ca0-47d2-8d8f-a4fe259f3b55)

Add UWP extension in UWP/Reference 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/c3a48b03-6178-4231-be48-334583b6ae3d)
 
Add and cover this in UWP/App.xmal.cs
```C#
sealed partial class App : Application
    {
        public static BackgroundTaskDeferral AppServiceDeferral = null;
        public static AppServiceConnection Connection = null;
        public static bool IsForeground = false;
        public static event EventHandler<AppServiceTriggerDetails> AppServiceConnected;
        public static event EventHandler AppServiceDisconnected;



        private void App_LeavingBackground(object sender, LeavingBackgroundEventArgs e)
        {
            IsForeground = true;
        }

        private void App_EnteredBackground(object sender, EnteredBackgroundEventArgs e)
        {
            IsForeground = false;
        }

        /// <summary>
        /// Initializes the singleton application object.  This is the first line of authored code
        /// executed, and as such is the logical equivalent of main() or WinMain().
        /// </summary>
        public App()
        {
            this.InitializeComponent();
            this.Suspending += OnSuspending;
            this.EnteredBackground += App_EnteredBackground;
            this.LeavingBackground += App_LeavingBackground;
        }

        /// <summary>
        /// Handles connection requests to the app service
        /// </summary>
        protected override void OnBackgroundActivated(BackgroundActivatedEventArgs args)
        {
            base.OnBackgroundActivated(args);

            if (args.TaskInstance.TriggerDetails is AppServiceTriggerDetails details)
            {
                // only accept connections from callers in the same package
                if (details.CallerPackageFamilyName == Package.Current.Id.FamilyName)
                {
                    // connection established from the fulltrust process
                    AppServiceDeferral = args.TaskInstance.GetDeferral();
                    args.TaskInstance.Canceled += OnTaskCanceled;

                    Connection = details.AppServiceConnection;
                    AppServiceConnected?.Invoke(this, args.TaskInstance.TriggerDetails as AppServiceTriggerDetails);
                }
            }
        }

        /// <summary>
        /// Task canceled here means the app service client is gone
        /// </summary>
        private void OnTaskCanceled(IBackgroundTaskInstance sender, BackgroundTaskCancellationReason reason)
        {
            AppServiceDeferral?.Complete();
            AppServiceDeferral = null;
            Connection = null;
            AppServiceDisconnected?.Invoke(this, null);
        }

....

```

* launch app and re-launch when disconnection
Add and cover in UWP/MainPage.xmal.cs
```C#
 public MainPage()
        {
            this.InitializeComponent();

           // ApplicationView.PreferredLaunchViewSize = new Size(200, 200);
           // ApplicationView.PreferredLaunchWindowingMode = ApplicationViewWindowingMode.PreferredLaunchViewSize;
        }

        protected async override void OnNavigatedTo(NavigationEventArgs e)
        {
            base.OnNavigatedTo(e);

            if (ApiInformation.IsApiContractPresent("Windows.ApplicationModel.FullTrustAppContract", 1, 0))
            {
                //for connect or disconnect event
                //App.AppServiceConnected += MainPage_AppServiceConnected;
                //App.AppServiceDisconnected += MainPage_AppServiceDisconnected;

                //for sideload app
                await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
            }
        }

        /// <summary>
        /// When the desktop process is disconnected, reconnect if needed
        /// </summary>
        private async void MainPage_AppServiceDisconnected(object sender, EventArgs e)
        {
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            { 
                Reconnect();
            });
        }

        /// <summary>
        /// Ask user if they want to reconnect to the desktop process
        /// </summary>
        private async void Reconnect()
        {
            if (App.IsForeground)
            {
                MessageDialog dlg = new MessageDialog("Connection to desktop process lost. Reconnect?");
                UICommand yesCommand = new UICommand("Yes", async (r) =>
                {
                    await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
                });
                dlg.Commands.Add(yesCommand);
                UICommand noCommand = new UICommand("No", (r) => { });
                dlg.Commands.Add(noCommand);
                await dlg.ShowAsync();
            }
        }

.......

```

* if can not find entrypoint, can see this for more information  
https://stackoverflow.com/questions/77065216/dep0700-registration-of-the-app-failed-while-uwp-extension-launch-consult-app/77493220#77493220


# Sideload app and UWP can change message in two way

Go to BelaunchedApp/reference and add those file 
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/9f198e02-c419-4764-a2b8-77e8e318e154)

* It can send data and receive data 
Add and cover sildload_app/program
```C#
class Program
    { 
        static private AppServiceConnection connection = null; 

        static void Main(string[] args)
        {
            InitializeAppServiceConnection();

            string s_res;

            Console.WriteLine("[1] send data to UWP");
            while ((s_res = Console.ReadLine()) != "")
            {
                int i_res=int.Parse(s_res);

                if (i_res == 1)
                {
                    Send2UWP_2("Hi!", "UWP");
                }
            }
        }

        /// <summary>
        /// Open connection to UWP app service
        /// </summary>
        static private async void InitializeAppServiceConnection()
        {
            connection = new AppServiceConnection();
            connection.AppServiceName = "SampleInteropService";
            connection.PackageFamilyName = Package.Current.Id.FamilyName;
            connection.RequestReceived += Connection_RequestReceived;
            connection.ServiceClosed += Connection_ServiceClosed;

            AppServiceConnectionStatus status = await connection.OpenAsync();
            if (status != AppServiceConnectionStatus.Success)
            {
                //In console, something wrong
                Console.WriteLine(status.ToString());
                //In app, something went wrong ...
                //MessageBox.Show(status.ToString());
                //this.IsEnabled = false;
            }
        }

        /// <summary>
        /// Handles the event when the desktop process receives a request from the UWP app
        /// </summary>
        static private async void Connection_RequestReceived(AppServiceConnection sender, AppServiceRequestReceivedEventArgs args)
        {
            // retrive the reg key name from the ValueSet in the request
            /*
            string key = args.Request.Message["KEY"] as string;
            if (key == "abcd")
            {
                // compose the response as ValueSet
                ValueSet response = new ValueSet();
                response.Add("GOOD", "KEY IS FOUNDED ^^");

                // send the response back to the UWP
                await args.Request.SendResponseAsync(response);
            }
            else
            {
                ValueSet response = new ValueSet();
                response.Add("ERROR", "INVALID REQUEST");
                await args.Request.SendResponseAsync(response);
            }
            */
            int? key1 = args.Request.Message["KEY1"] as int?;
            int? key2 = args.Request.Message["KEY2"] as int?;

            if (key1 != null && key2 != null)
            {
                int ans = (int)key1 + (int)key2;

                // compose the response as ValueSet
                ValueSet response = new ValueSet();
                response.Add("KEY3", ans);

                // send the response back to the UWP
                await args.Request.SendResponseAsync(response);
            }
        }

        /// <summary>
        /// Handles the event when the app service connection is closed
        /// </summary>
        static private void Connection_ServiceClosed(AppServiceConnection sender, AppServiceClosedEventArgs args)
        {
            //In console, connection to the UWP lost, so we shut down the desktop process
            Console.WriteLine("UWP lost connection! Please restart.");
            Console.ReadLine();
            Environment.Exit(0);
            //In app, connection to the UWP lost, so we shut down the desktop process
            //Dispatcher.BeginInvoke(DispatcherPriority.Normal, new Action(() =>
            //{
            //    Application.Current.Shutdown();
            //}));
        }

        static private async void Send2UWP(double a, double b)
        {
            // ask the UWP to calculate d1 + d2
            ValueSet request = new ValueSet();
            request.Add("D1", a);
            request.Add("D2", b);

            //start sending
            AppServiceResponse response = await connection.SendMessageAsync(request);
            //get response
            double result = (double)response.Message["RESULT"];

            Console.WriteLine(result.ToString());
        }

        static private async void Send2UWP_2(string a, string b)
        {
            // ask the UWP to calculate d1 + d2
            ValueSet request = new ValueSet();
            request.Add("s_a", a);
            request.Add("s_b", b);

            //start sending
            AppServiceResponse response = await connection.SendMessageAsync(request);
            //get response
            string result = response.Message["toConsole_result"] as string;

            Console.WriteLine("send data_a to UWP: " + a);
            Console.WriteLine("send data_b to UWP: " + b);
            Console.WriteLine("getting data from UWP: " + result);
        }
    }
```

Add and cover UWP/MainPage.xmal 
```XML
<Grid>
        <Button  x:Name="calc_btn"   Margin="49,100,0,0" VerticalAlignment="Top" Click="calc_btn_Click" Height="58" Width="117">
            <TextBlock>calculate<LineBreak/>in sideload app</TextBlock>
        </Button>
        <TextBlock x:Name="puse_textBox" HorizontalAlignment="Left" Margin="124,42,0,0" Text="+" TextWrapping="Wrap" VerticalAlignment="Top" Width="33"/>
        <TextBox x:Name="a_textBlock" HorizontalAlignment="Left" Margin="44,38,0,0" Text="10" TextWrapping="Wrap" VerticalAlignment="Top"/>
        <TextBox x:Name="b_textBlock" HorizontalAlignment="Left" Margin="164,38,0,0" Text="20" TextWrapping="Wrap" VerticalAlignment="Top"/>
        <TextBlock x:Name="equal_textBox" HorizontalAlignment="Left" Margin="244,42,0,0" Text="=" TextWrapping="Wrap" VerticalAlignment="Top" Width="33"/>
        <TextBox x:Name="ans_textBox" HorizontalAlignment="Left" Margin="285,15,0,0" Text="" TextWrapping="Wrap" VerticalAlignment="Top" Width="238" Height="100"/>
        <TextBox x:Name="textBox" HorizontalAlignment="Left" Margin="54,197,0,0" Text="TextBox" TextWrapping="Wrap" VerticalAlignment="Top"/>

</Grid>
```

* It can send data and receive data
Add and cover UWP/MainPage.xmal.cs
```C#
/// <summary>
    /// An empty page that can be used on its own or navigated to within a Frame.
    /// </summary>
    public sealed partial class MainPage : Page
    {
        public MainPage()
        {
            this.InitializeComponent();

            // ApplicationView.PreferredLaunchViewSize = new Size(200, 200);
            // ApplicationView.PreferredLaunchWindowingMode = ApplicationViewWindowingMode.PreferredLaunchViewSize;
        }

        protected async override void OnNavigatedTo(NavigationEventArgs e)
        {
            base.OnNavigatedTo(e);

            if (ApiInformation.IsApiContractPresent("Windows.ApplicationModel.FullTrustAppContract", 1, 0))
            {
                //for connect or disconnect event
                App.AppServiceConnected += MainPage_AppServiceConnected;
                App.AppServiceDisconnected += MainPage_AppServiceDisconnected;

                //ApplicationData.Current.LocalSettings.Values["parameters"] = "test";

                //for sideload app
                await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
            }
        }


        /// <summary>
        /// When the desktop process is connected, get ready to send/receive requests
        /// </summary>
        private async void MainPage_AppServiceConnected(object sender, AppServiceTriggerDetails e)
        {
            App.Connection.RequestReceived += AppServiceConnection_RequestReceived;
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            {
                // enable UI to access  the connection
                //btnRegKey.IsEnabled = true;
            });
        }

        /// <summary>
        /// Handle calculation request from desktop process
        /// (dummy scenario to show that connection is bi-directional)
        /// </summary>
        private async void AppServiceConnection_RequestReceived(AppServiceConnection sender, AppServiceRequestReceivedEventArgs args)
        {
            //get value from sideload app
            string s_a = (string)args.Request.Message["s_a"];
            string s_b = (string)args.Request.Message["s_b"];
            //double result = d1 + d2;

            //send "have get data" to sideload app
            ValueSet response = new ValueSet();
            response.Add("toConsole_result", "Hello! sideload app");
            await args.Request.SendResponseAsync(response);

            //textBox.Text = ""; //it will error
            //log the getting value in the UI for demo purposes
            await this.Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>     //I don't know this code
            {
                textBox.Text += string.Format("Request(getting data from sideload app):\ndata 1: {0} \ndata 2: {1}", s_a, s_b);
            });
        }


        /// <summary>
        /// When the desktop process is disconnected, reconnect if needed
        /// </summary>
        private async void MainPage_AppServiceDisconnected(object sender, EventArgs e)
        {
            await Dispatcher.RunAsync(CoreDispatcherPriority.Normal, () =>
            {
                Reconnect();
            });
        }

        /// <summary>
        /// Ask user if they want to reconnect to the desktop process
        /// </summary>
        private async void Reconnect()
        {
            if (App.IsForeground)
            {
                MessageDialog dlg = new MessageDialog("Connection to desktop process lost. Reconnect?");
                UICommand yesCommand = new UICommand("Yes", async (r) =>
                {
                    await FullTrustProcessLauncher.LaunchFullTrustProcessForCurrentAppAsync();
                });
                dlg.Commands.Add(yesCommand);
                UICommand noCommand = new UICommand("No", (r) => { });
                dlg.Commands.Add(noCommand);
                await dlg.ShowAsync();
            }
        }

        private async void calc_btn_Click(object sender, RoutedEventArgs e)
        {
            //send value to sideloade app
            ValueSet request = new ValueSet();
            request.Add("KEY1", int.Parse(a_textBlock.Text));
            request.Add("KEY2", int.Parse(b_textBlock.Text));
            AppServiceResponse response = await App.Connection.SendMessageAsync(request);   //send data and get response 

            //display the response key/value pairs
            //tbResult.Text = "";
            foreach (string key in response.Message.Keys)
            {
                ans_textBox.Text = "sended by sideload app\nkey: " + key + "\nvalue: " + response.Message[key];
            }
        }
    }
```


# package to app
* TODO...


* And should generate self-signed certification.  
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/0aca610e-2e1f-483e-be29-ae0ea8851638)

* Add references Windows Desktop Extensions for the UWP in UWP project.  
![image](https://github.com/testtestProblem/UWP_extention/assets/107662393/9c2721f2-0fc4-4a04-8162-bc3da3b69213)


# Other

error code => DEP1000: Cannot copy file 
Just close all debug window
![image](https://github.com/testtestProblem/UWP_Extension_Example/assets/107662393/1654822f-431d-4822-87f5-7c9279c98887)


# Reference
* https://stefanwick.com/2018/04/16/uwp-with-desktop-extension-part-3/
* https://www.cnblogs.com/manupstairs/p/14178931.html
* https://github.com/StefanWickDev/UWP-FullTrust/tree/master/UWP_FullTrust_3
* https://freetion26.medium.com/uwp%E9%80%8F%E9%81%8Eappserviceconnection%E5%92%8Cwin32-process%E6%BA%9D%E9%80%9A%E5%8F%96%E5%BE%97%E7%B3%BB%E7%B5%B1%E8%B7%AF%E5%BE%91%E4%B8%8B%E6%AA%94%E6%A1%88%E5%88%97%E8%A1%A8-981dd0765e52
