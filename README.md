# audio_service

Play audio in the background.

* Continues playing while the screen is off or the app is in the background
* Control playback from your flutter UI, headest, Wear OS or Android Auto
* Drive audio playback from custom Dart code

This plugin provides a framework for playing arbitrary audio in the background (e.g. audio files, or even text-to-speech). For Android, this means acquiring a wake lock so that it will run even with the screen turned off, acquiring audio focus so that you can control your app from a headset, creating a media session and media browser service so that your app can be controlled by wearable devices and Android Auto. The plugin provides all of this while you simply supply the dart code to drive audio playback.

Since you can write the audio playback code in dart, you are free to use any other flutter audio plugin in conjunction with this one to play whatever audio you want. There are already a number of flutter plugins for playing audio files or synthesising text to speech, however since background execution of dart code is a relatively new feature of flutter, not all of those plugins are yet compatible (I am contacting some of these authors to make their packages compatible.)

Only the Android side is implemented for now. iOS contributors are welcome to help complete the iOS side (see the bottom of this page for details).

## Example

Client-side code:

```dart
await AudioService.start(
	backgroundTask: myBackgroundTask,
	notificationChannelName: 'Music Player',
	notificationColor: 0xFF2196f3,
	androidNotificationIcon: "mipmap/ic_launcher",
);
```

Background code:

```dart
void myBackgroundTask() {
	Completer completer = Completer();
	
	AudioServiceBackground.run(
		doTask: () async {
			// play audio (custom code)
			// ...
			// ...
			await completer.future;
			// Background execution stops as soon as doTask completes
		},
		onStop: () {
			completer.complete(); // one way to control completion
		},
		onClick: (int eventTime, MediaButton button) {
			// custom code
		},
	);
}
```

Android manifest file:

```xml
<manifest ...>
	<uses-permission android:name="android.permission.WAKE_LOCK"/>
	<application
		android:name=".MainApplication"
		...>
		
		...
		
		<service android:name="com.ryanheise.audioservice.AudioService">
			<intent-filter>
				<action android:name="android.media.browse.MediaBrowserService" />
			</intent-filter>
		</service>

		<receiver android:name="android.support.v4.media.session.MediaButtonReceiver" >
			<intent-filter>
				<action android:name="android.intent.action.MEDIA_BUTTON" />
			</intent-filter>
		</receiver> 
	</application>
</manifest>
```

Application class:

```java
public class MainApplication extends FlutterApplication implements PluginRegistry.PluginRegistrantCallback {
	@Override
	public void onCreate() {
		super.onCreate();
		AudioServicePlugin.setPluginRegistrantCallback(this);
	}

	@Override
	public void registerWith(PluginRegistry registry) {
		GeneratedPluginRegistrant.registerWith(registry);
	}
}
```

## Help/Contribute

If you know how to implement any of these features in iOS, pull requests are welcome! As a guideline, prefer to keep the same dart API for both Android and iOS where possible. In cases where there are unavoidable differences between Android and iOS, name the feature with an `android` or `ios` prefix. 

If you find a flutter plugin (audio or otherwise) that crashes when running in the background environment, another way you can help is to file a bug report with that project, letting them know of the simple fix to make it work (see below).

### Sample bug report

Below is a sample bug report I submitted to the `wifi` plugin project.

Flutter's new background execution feature (described here: https://medium.com/flutter-io/executing-dart-in-the-background-with-flutter-plugins-and-geofencing-2b3e40a1a124) allows plugins to be registered in a background context (e.g. a Service). The problem is that the wifi plugin assumes that the context for plugin registration is an activity with this line of code:

`		WifiManager wifiManager = (WifiManager) registrar.activity().getApplicationContext().getSystemService(Context.WIFI_SERVICE);`

`registrar.activity()` may now return null, and this leads to a `NullPointerException`:

```
E/AndroidRuntime( 2453): 	at com.ly.wifi.WifiPlugin.registerWith(WifiPlugin.java:23)
E/AndroidRuntime( 2453): 	at io.flutter.plugins.GeneratedPluginRegistrant.registerWith(GeneratedPluginRegistrant.java:30)
```

The solution is to change the above line of code to this:

`		WifiManager wifiManager = (WifiManager) registrar.activeContext().getApplicationContext().getSystemService(Context.WIFI_SERVICE);`
