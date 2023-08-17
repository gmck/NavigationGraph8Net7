# NavigationGraph8Net7 : net7.0-android33
**August 17, 2023**

After some more experimenting I have come up with a simpler method of stopping the dummy service. In the previous version we saw from the LogCat logs that we needed to call Activity.Finish() within OnActivitySaveInstanceState of the Application class so that when MainActivty.OnDestroy() was reached IsFinishing would be true so that our dummy StopService method would be called. This was necessary because the OnActivityDestroyed was not being called when the Developer Options Apps - Don’t keep activities was set to OFF. That being the default state for a normal user, it was necessary to find a way to stop the service when the app was exited.

From the charts provided in Google's documentation - https://developer.android.com/guide/components/activities/intro-activities we can see that Activity's OnDestroy states - **The system invokes this callback before an activity is destroyed**. 

However, in a Single Activity application using the NavigationComponent we can see that this is not true. If you put a breakpoint on OnDestroy of the MainActivity you'll see that the only time that breakpoint is hit is when the device is rotated. When exiting the app via a back gesture, the debugger does not break at the MainActivity's OnDestroy, the app just ends. This can be partially explained when you look further into the documentation under **Processes and App Lifecycle** with the following information ***Be aware that onDestroy() is not guaranteed to be called in the case that a process is killed by the system.*** Typical Google documentation, mentioned but not expanded on. It does though perfectly describe this example.

In this update we now have an OnSaveInstanceState override in the MainActivity. This is called when the app is exited via a back gesture. 

```c#
protected override void OnSaveInstanceState(Bundle outState)
{
    base.OnSaveInstanceState(outState);

    if (!IsChangingConfigurations)
    {
        Finish();
        Log.Debug(logTag, logTag + " OnSaveInstanceState - Calling Finish()");
    }
}
```

So we can dispense with the OnActivitySaveInstanceState override in the Application class or even the whole class. I've commented out the RegisterActivityLifecycleCallbacks(this) and the UnregisterActivityLifecycleCallbacks(this) in the Application class so the callbacks are no longer executed.

The end result is the Predictive Back Gesture works as before and StopService is called.

Please note that we can still see the following in the LogCat logs, but since they are only warnings I think it is probably safe to ignore them.

Input channel object '60d57a Splash Screen com.companyname.navigationgraph8net7 (client)' was disposed without first being removed with the input manager!

Input channel object '60a1c5d com.companyname.navigationgraph8net7/crc640cebeb752226fc06.MainActivity (client)' was disposed without first being removed with the input manager!

**August 13, 2023**


NavigationGraph8Net7 is a replacement for NavigationGraph7Net7. 

Since developing NavigationGraph7Net7 I found that I was using a Developer Options setting that badly affected the behavior of the app when exiting the app.
For a long time, I’ve had the Developer Options Don’t Keep Activities in the On position on the devices I deploy to. This has the effect of affecting the code that is executed in the Application class, NavigationGraph7ApplicationNet7.cs. In the table below you can see from the LogCat logs that the code in OnActivityDestroyed is not executed if the Don’t keep activities is set to OFF, so consequently NavigationGraph7Net7 was not calling activity.Finish() and therefore my dummy function StopService was never called. As it is a dummy function, no real harm is done with this app, but when you use that code in a real app that contains a bound service then obviously that service is not going to be stopped.

Thankfully I was only getting around to using this code in my main application, so this bug has never made it to a production app, but of course, it may have confused anyone who happened to download and study NavigationGraph7Net7 if they had stepped through the code in the debugger. 

I only stumbled upon it because I decided to rebuild a Pixel4a by wiping it, as I was encountering problems with a particular email address when testing Google Subscriptions code with it. After rebuilding it I set up Developer Options again but didn’t change the default setting of **Don’t keep Activities**. On testing my app again I found this weird problem where every time I exited the app, the bound service didn’t stop. In this app, I display a permanent notification, which also places an icon in the status bar indicating that the service is running. On quitting the app, the icon should disappear. However, it didn’t and therefore the service was still running. It took a while, but I eventually got around to checking if the rebuilt Pixel4a was different in any way from my other devices. Then I remembered I hadn’t set Don’t Keep Activities to On.

From the very beginning of using the NavigationComponent, I’d always had problems getting IsFinishing == true. The original fix was to use an OnBackPressedCallback in the HomeFragment. Then use its HandleOnBackPressed method to call Activity.Finish(). That worked well and the service was closed from within the MainActivity’s OnDestroy() where you then would get IsFinishing == true and then call StopService. 

However, we then had to add support for the Predictive Back Gesture. The PBG requires that there should be no enabled OnBackPressedCallback in the HomeFragment or Start Destination Fragment for the PBG to appear. So the onBackPressedCallback was removed from the HomeFragment and that then led to the solution of using the Application Class’s Application.IActivityLifecycleCallbacks, and to the code snippet in the table below using OnActivityDestroyed, which again made IsFinishing == true in the OnDestroy of the MainActivity and StopService was again successfully called.

In NavigationGraph7Net7, back in June, I had already commented out all the OnBackPressedCallbacks from each fragment in anticipation of what is likely in Android 14 as I believe as you close each fragment there will also be some sort of animation like a PBG and it will probably also be foiled by any OnBackPressedCallbacks. 

To maintain the same functionality as the callbacks provide I modified the NavOptions code in each of OnNavigationItemSelected, BottomNavigationView_ItemSelected in the MainActivity and the OnMenuItemSelected in the HomeFragment. The modification is very simple as in added SetPopUpTo(Resource.Id.home_fragment, false, true) to NavOptions for the OnNavigationItemSelected and the OnMenuItemSelected and SetPopUpTo(Resource.Id.slideshow_fragment, false, true) for the BottomNavigationView_ItemSelected.    

Last March the Predictive Back Gesture was broken for all Android 13 Pixel devices by the March update so I therefore also made android:enableOnBackInvokedCallback="false" in the application tag of the AndroidManifest. The PBG was still working on Samsung devices, so those users, could flick it back to true.

I’ve left NavigationGraph7Net7 as is and have now created a new NavigationGraph8Net7 project. NavigationGraph8Net7 has all the code that was commented out in NavigationGraph7Net7 removed from the project and the NavFragmentOnBackPressedCallback.cs file has also been deleted. 

The August update for Android 13 devices has fixed the Predictive Back Gesture, so NavigationGraph8Net7 restores the previous behaviour that was originally present in NavigationGraph7Net7, both the PBG and the dummy StopService method work again. 

I checked with the Android Issue tracker, and note that the report is still unresolved. https://issuetracker.google.com/issues/275597731. Google is warning developers that back navigation will be broken in their apps if they don’t support this feature when it’s enforced.


**Developer Options Apps - Don’t keep activities 	ON**

App has the following code in OnActivityDestroyed
```c#
if (!activity.IsChangingConfigurations)
{
	activity.Finish();
	Log.Debug(logTag, logTag + " OnActivityDestroyed - Calling Activity.Finish()");
}
```

***Results***

**Start App**

```
Nav-Application OnCreate - RegisterActivityLifecycleCallbacks
Nav-Application OnActivityCreated
Nav-Application OnActivityStarted
```
**Exit App**

```
Nav-Application OnActivityResumed
Nav-Application OnActivityPaused
Nav-Application OnActivtyStopped
Nav-Application OnActivitySaveInstanceState
Nav-Application OnActivityDestroyed - Calling Activity.Finish()
Nav-HomeFragment OnDestroy
Nav-MainActivity OnDestroy IsFinishing is True
Nav-MainActivity StopService - called from OnDestroy
```

**Developer Options Apps - Don’t keep activities 	OFF**

**Start App**

```
Nav-Application OnCreate - RegisterActivityLifecycleCallbacks
Nav-Application OnActivityStarted
Nav-Application OnActivityResumed
```

**Exit App**

```
Nav-Application OnActivityPaused
Nav-Application OnActivtyStopped
Nav-Application OnActivitySaveInstanceState - Calling Activity.Finish()
Nav-HomeFragment OnDestroy
Nav-MainActivity OnDestroy IsFinishing is True
Nav-MainActivity StopService - called from OnDestroy
```

This is the OFF version, which doesn't receive an OnActivityDestroyed event, but does receive an OnActivitySaveInstanceState event, so we place the same code from the OnActivityDestroyed event in the OnActivitySaveInstanceState event to achieve the same result. 

Please note that you also need to have Predictive Back Gesture enabled in Developers Options for the PBG to appear and the August update applied to any Pixel device running Android 13.

To see the output of these logs in Android Studio just set the filter to tag:nav





