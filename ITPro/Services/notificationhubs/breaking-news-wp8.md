<properties linkid="develop-notificationhubs-tutorials-send-breaking-news-wp8" writer="glenga" urlDisplayName="Breaking News" pageTitle="Notification Hubs Breaking News Tutorial" title="Notification Hubs Breaking News Tutorial" metaKeywords="" description="Learn how to use Windows Azure Service Bus Notification Hubs to send breaking news notifications." metaCanonical="" disqusComments="1" umbracoNaviHide="1" />

# Use Notification Hubs to send breaking news
<div class="dev-center-tutorial-selector sublanding"> 
    	<a href="/en-us/manage/services/notification-hubs/breaking-news-dotnet" title="Windows Store C#">Windows Store C#</a>
		<a href="/en-us/manage/services/notification-hubs/breaking-news-wp8" title="Windows Phone" class="current">Windows Phone</a>
		<a href="/en-us/manage/services/notification-hubs/breaking-news-ios" title="iOS">iOS</a>
</div>

This topic shows you how to use Windows Azure Notification Hubs to broadcast breaking news notifications to a Windows Phone app. When complete, you will be able to register for breaking news categories you are interested in, and receive only push notifications for those categories. This scenario is a common pattern for many apps where notifications have to be sent to groups of users that have previously declared interest in them, e.g. RSS reader, apps for music fans, etc. 

Broadcast scenarios are enabled by including one or more _tags_ when creating a registration in the notification hub. When notifications are sent to a tag, all devices that have registered for the tag will receive the notification. Because tags are simply strings, they do not have to be provisioned in advance. For more information about tags, refer to [Notification Hubs Guidance]. 

This tutorial walks you through these basic steps to enable this scenario:

1. [Add category selection to the app]
2. [Register for notifications]
3. [Send notifications from your back-end]
4. [Run the app and generate notifications]

This topic builds on the app you created in [Get started with Notification Hubs]. Before starting this tutorial, you must have already completed [Get started with Notification Hubs].

##<a name="adding-categories"></a>Add category selection to the app

The first step is to add the UI elements to your existing main page that enable the user to select categories to register. The categories selected by a user are stored on the device. When the app starts, a device registration is created in your notification hub with the selected categories as tags. 

1. Open the MainPage.xaml project file, then replace the **Grid** elements named `TitlePanel` and `ContentPanel` with the following code:
			
        <StackPanel x:Name="TitlePanel" Grid.Row="0" Margin="12,17,0,28">
            <TextBlock Text="Breaking News" Style="{StaticResource PhoneTextNormalStyle}" Margin="12,0"/>
            <TextBlock Text="Categories" Margin="9,-7,0,0" Style="{StaticResource PhoneTextTitle1Style}"/>
        </StackPanel>

        <Grid Name="ContentPanel" Grid.Row="1" Margin="12,0,12,0">
            <Grid.RowDefinitions>
                <RowDefinition Height="auto"/>
                <RowDefinition Height="auto" />
                <RowDefinition Height="auto" />
                <RowDefinition Height="auto" />
            </Grid.RowDefinitions>
            <Grid.ColumnDefinitions>
                <ColumnDefinition />
                <ColumnDefinition />
            </Grid.ColumnDefinitions>
            <CheckBox Name="WorldCheckBox" Grid.Row="0" Grid.Column="0">World</CheckBox>
            <CheckBox Name="PoliticsCheckBox" Grid.Row="1" Grid.Column="0">Politics</CheckBox>
            <CheckBox Name="BusinessCheckBox" Grid.Row="2" Grid.Column="0">Business</CheckBox>
            <CheckBox Name="TechnologyCheckBox" Grid.Row="0" Grid.Column="1">Technology</CheckBox>
            <CheckBox Name="ScienceCheckBox" Grid.Row="1" Grid.Column="1">Science</CheckBox>
            <CheckBox Name="SportsCheckBox" Grid.Row="2" Grid.Column="1">Sports</CheckBox>
            <Button Name="SubscribeButton" Content="Subscribe" HorizontalAlignment="Center" Grid.Row="3" Grid.Column="0" Grid.ColumnSpan="2" Click="SubscribeButton_Click" />
        </Grid>

2. In the project, create a new class named **Notifications**, add the **public** modifier to the class definition, then add the following **using** statements to the new code file:

		using Microsoft.Phone.Notification;
		using Microsoft.WindowsAzure.Messaging;
		using System.IO.IsolatedStorage;

3. Copy the following code into the new **Notifications** class:

		private NotificationHub hub;

		public Notifications()
		{
		    hub = new NotificationHub("<hub name>", "<connection string with listen access>");
		}
		
		public async Task StoreCategoriesAndSubscribe(IEnumerable<string> categories)
		{
		    var categoriesAsString = string.Join(",", categories);
		    var settings = IsolatedStorageSettings.ApplicationSettings;
		    if (!settings.Contains("categories"))
		    {
		        settings.Add("categories", categoriesAsString);
		    }
		    else
		    {
		        settings["categories"] = categoriesAsString;
		    }
		    settings.Save();
		
		    await SubscribeToCategories(categories);
		}
		
		public async Task SubscribeToCategories(IEnumerable<string> categories)
		{
		    var channel = HttpNotificationChannel.Find("MyPushChannel");
		
		    if (channel == null)
		    {
		        channel = new HttpNotificationChannel("MyPushChannel");
		        channel.Open();
		        channel.BindToShellToast();
		    }
		
		    await hub.RegisterNativeAsync(channel.ChannelUri.ToString(), categories);
		}

    This class uses the local storage to store the categories of news that this device has to receive. It also contains methods to register for these categories.

4. In the above code, replace the `<hub name>` and `<connection string with listen access>` placeholders with your notification hub name and the connection string for *DefaultListenSharedAccessSignature* that you obtained earlier.

	<div class="dev-callout"><strong>Note</strong> 
		<p>Because credentials that are distributed with a client app are not generally secure, you should only distribute the key for listen access with your client app. Listen access enables your app to register for notifications, but existing registrations cannot be modified and notifications cannot be sent. The full access key is used in a secured backend service for sending notifications and changing existing registrations.</p>
	</div> 

4. In the App.xaml.cs project file, add the following property to the **App** class:

		public Notifications notifications = new Notifications();

	This property is used to create and access a **Notifications** instance.

5. In your MainPage.xaml.cs, add the following line:

		using Windows.UI.Popups;

6. In the MainPage.xaml.cs project file, add the following method:

		private async void SubscribeButton_Click(object sender, RoutedEventArgs e)
		{
		    var categories = new HashSet<string>();
		    if (WorldCheckBox.IsChecked == true) categories.Add("World");
		    if (PoliticsCheckBox.IsChecked == true) categories.Add("Politics");
		    if (BusinessCheckBox.IsChecked == true) categories.Add("Business");
		    if (TechnologyCheckBox.IsChecked == true) categories.Add("Technology");
		    if (ScienceCheckBox.IsChecked == true) categories.Add("Science");
		    if (SportsCheckBox.IsChecked == true) categories.Add("Sports");
		
		    await ((App)Application.Current).notifications.StoreCategoriesAndSubscribe(categories);
		
		    MessageBox.Show("Subscribed to: " + string.Join(",", categories));
		}
	
	This method creates a list of categories and uses the **Notifications** class to store the list in the local storage and register the corresponding tags with your notification hub. When categories are changed, the registration is recreated with the new categories.

Your app is now able to store a set of categories in local storage on the device and register with the notification hub whenever the user changes the selection of categories. 

##<a name="register"></a>Register for notifications

These steps register with the notification hub on startup using the categories that have been stored in local storage. 

<div class="dev-callout"><strong>Note</strong> 
	<p>Because there is no guarantee that the channel used by a given device is persisted, you should register for notifications frequently to avoid notification failures. This example registers for notification every time that the app starts. For apps that are run frequently, more than once a day, you can probably skip registration to preserve bandwidth if less than a day has passed since the previous registration.</p>
</div> 

1. Add the following code to the **Notifications** class:

		public IEnumerable<string> RetrieveCategories()
		{
		    var categories = (string)IsolatedStorageSettings.ApplicationSettings["categories"];
		    return categories != null ? categories.Split(',') : new string[0];
		}

	This returns the categories defined in the class.

1. Open the App.xaml.cs file and add the **async** modifier to **Application_Launching** method.

2. In the **Application_Launching** method, locate and replace the Notification Hubs registration code that you added in [Get started with Notification Hubs] with the following line of code:

		await notifications.SubscribeToCategories(notifications.RetrieveCategories());

	This makes sure that every time the app starts it retrieves the categories from local storage and requests a registeration for these categories. 

3. In the MainPage.xaml.cs project file, add the following code that implements the **OnNavigatedTo** method:

		protected override void OnNavigatedTo(NavigationEventArgs e)
		{
		    var categories = ((App)Application.Current).notifications.RetrieveCategories();
		
		    if (categories.Contains("World")) WorldCheckBox.IsChecked = true;
		    if (categories.Contains("Politics")) PoliticsCheckBox.IsChecked = true;
		    if (categories.Contains("Business")) BusinessCheckBox.IsChecked = true;
		    if (categories.Contains("Technology")) TechnologyCheckBox.IsChecked = true;
		    if (categories.Contains("Science")) ScienceCheckBox.IsChecked = true;
		    if (categories.Contains("Sports")) SportsCheckBox.IsChecked = true;
		}

	This updates the main page based on the status of previously saved categories. 

The app is now complete and can store a set of categories in the device local storage used to register with the notification hub whenever the user changes the selection of categories. Next, we will define a backend that can send category notifications to this app.

<h2><a name="send"></a><span class="short-header">Send notifications</span>Send notifications from your back-end</h2>

<div chunk="../chunks/notification-hubs-back-end.md" />

##<a name="test-app"></a>Run the app and generate notifications

1. In Visual Studio, press F5 to compile and start the app.

	![][1] 

	Note that the app UI provides a set of toggles that lets you choose the categories to subscribe to. 

2. Enable one or more categories toggles, then click **Subscribe**.

	The app converts the selected categories into tags and requests a new device registration for the selected tags from the notification hub. The registered categories are returned and displayed in a dialog.

	![][19]

4. Send a new notification from the backend in one of the following ways:

	+ **Console app:** start the console app.

	+ **Mobile Services:** click the **Scheduler** tab, click the job, then click **Run once**.

	Notifications for the selected categories appear as toast notifications.

	![][14]

You have completed this topic.

<!--## <a name="next-steps"> </a>Next steps

In this tutorial we learned how to broadcast breaking news by category. Consider completing one of the following tutorials that highlight other advanced Notification Hubs scenarios:

+ [Use Notification Hubs to broadcast localized breaking news]

	Learn how to expand the breaking news app to enable sending localized notifications. 

+ [Notify users with Notification Hubs]

	Learn how to push notifications to specific authenticated users. This is a good solution for sending notifications only to specific users.
-->

<!-- Anchors. -->
[Add category selection to the app]: #adding-categories
[Register for notifications]: #register
[Send notifications from your back-end]: #send
[Run the app and generate notifications]: #test-app
[Next Steps]: #next-steps

<!-- Images. -->
[1]: ../media/notification-hub-breakingnews-win1.png
[13]: ../media/notification-hub-create-console-app.png
[14]: ../media/notification-hub-windows-toast-2.png
[15]: ../media/notification-hub-scheduler1.png
[16]: ../media/notification-hub-scheduler2.png
[19]: ../media/notification-hub-windows-reg-2.png

<!-- URLs.-->
[Get started with Notification Hubs]: /en-us/manage/services/notification-hubs/get-started-notification-hubs-wp8/
[Use Notification Hubs to broadcast localized breaking news]: ./breakingnews-localized-wp8.md 
[Notify users with Notification Hubs]: ./tutorial-notify-users-mobileservices.md
[Mobile Service]: ../../../DevCenter/Mobile/Tutorials/mobile-services-get-started.md
[Notification Hubs Guidance]: http://msdn.microsoft.com/en-us/library/jj927170.aspx
[Notification Hubs How-To for Windows Phone]: ??
[WindowsAzure.com]: http://www.windowsazure.com/
[Windows Azure Management Portal]: https://manage.windowsazure.com/




