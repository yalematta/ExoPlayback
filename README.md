# ExoPlayback
Introduction to Media Playback using ExoPlayer

### _Player UI Structure_

- Media Player takes some Media and render it as **Audio** or **Video**
- Media Controller includes all of the Playback buttons

_The Android Framework provides 2 classes:_

- Media Session 
- Media Controller 

They communicate with each other so that the UI stays _in sync_ with the Player using predefined _callbacks_: play, pause, stop etc. Think of it as a _Standard Decoupling Pattern_, the Media Session isolates the Player from the UI:

- So you can easily swap in different players without affecting the UI. 
- Or change the look & feel without changing the Player's features. 

It's a _Client Server Architecture_:

- The Media Session becomes the Server that holds the info about the Media and the state of the Player 
- Each controller becomes the Client that needs to sustain sync with the Media Session

### _Audio/Video_
Components of a Media Application can be implemented in 1 Activity:

- Media Controller hooked to the UI 
- Media Session which controls the Player

### _Audio Apps_
If the user navigates away:

- The Player may outlive the Activity that started it
- Media Session should run into a _Media Browsing Service_ which updates the UI when the Player state changes
- The Activity-Service communication is simplified by some framework classes

### _Comparing Players_

#### MediaPlayer
 - The basic functionality for a bared boned Player
 - Supports the most common Audio and Video formats
 - Supports very little customizations
 - Very straight forward to use
 - Good enough for many simple use cases

#### ExoPlayer
 - An Open Source Library that exposes the lower level Android Audio APIs
 - Supports High performance features like Dash, HLS...
 - You can customize the ExoPlayer code making it easy to add new components
 - Can only be used with Android version 4.1 or higher

#### Youtube Player
 - If you are playing videos specifically from Youtube, check the [YouTube Developer APIs](https://developers.google.com/youtube/)

#### Custom Player
- You can build a Custom Media Player from the low level Media API like [MediaCodec](https://developer.android.com/reference/android/media/MediaCodec.html), [AudioTrack](https://developer.android.com/reference/android/media/AudioTrack.html), and [MediaDrm](https://developer.android.com/reference/android/media/MediaDrm.html)

_ExoPlayer is the preferred choice:_

- It supports many different formats and extensible features
- It's a library you include in you application APK 
- You have control over which version you use
- You can easily update to a newer version as part of a regular application update

### _Add ExoPlayer_
- Media Player belongs in a Service or an Activity: depending on wether the App requires Background PlayBack
- Here we expect the user to be in the Activity when they are listening to the music
- We will implement our MediaPlayer inside the MainActivity.

### _Add SimpleExoPlayerView_
In your layout add the following:

```xml, [.highlight: 1]
<com.google.android.exoplayer2.ui.SimpleExoPlayerView
	android:id="@+id/playerView"
	android:layout_width="match_parent"
	android:layout_height="wrap_content" />
```

- Initialize the View in OnCreate
- Load a Background Image in the Player
- Create a SimpleExoPlayer instance by calling _initializePlayer_

```java
mPlayerView = (SimpleExoPlayerView) findViewById(R.id.playerView);
mPlayerView.setDefaultArtwork(BitmapFactory.decodeResource(getResources(), R.drawable.player_image)); 
initializePlayer(Uri.parse(answerSample.getUri()));

private void initializePlayer(Uri mediaUri){
	if (mExoPlayer == null){
		//Create an instance of the ExoPlayer
		TrackSelector trackSelector = new DefaultTrackSelector();
		LoadControl loadControl = new DefaultLoadControl(); 
		mExoPlayer = ExoPlayerFactory.newSimpleInstance(this, trackSelector, loadControl);
		mPlayerView.setPlayer(mExoPlayer);
		//Prepare the MediaSource
		String userAgent = Util.getUserAgent(this, "ClassicalMusicQuiz");
		MediaSource mediaSource = new ExtractorMediaSource(mediaUri, new DefaultDataSourceFactory(
			this, userAgent), new DefaultExtractorsFactory(), null, null);
		mExoPlayer.prepare(mediaSource);
		mExoPlayer.setPlayWhenReady(true);
	}
}
```
- Finally we want to stop and release the Player when the Activity is destroyed or also on OnPause or onStop, if you don't want the music to continue playing when the app is not visible.
- We will leave the music playing while the user might be checking other apps, but we do run the risk that the Activity might be destroyed by the System and therefore unexpectedly terminating PlayBack.

```java
@Override 
protected void onDestroy(){
	super.onDestroy();
	releasePlayer();
}

private void releasePlayer(){
	mExoPlayer.stop();
	mExoPlayer.release();
	mExoPlayer = null;
}
```
### _Customizing ExoPlayer UI_

#### ExoPlayer comes with 2 notable UI elements:

- PlaybackControlView is a view for controlling ExoPlayer instances. It displays standard playback controls including a play/pause button, fast-forward and rewind buttons, and a seek bar.
- SimpleExoPlayerView is a high level view for SimpleExoPlayer media playbacks. It displays video (or album art) and displays playback controls using a PlaybackControlView.

#### Overriding layout files

- SimpleExoPlayerView has a layout called: [exo_simple_player_view.xml](https://github.com/google/ExoPlayer/blob/release-v2/library/ui/src/main/res/layout/exo_simple_player_view.xml) 
- This layout file includes a PlayBackControlView which also uses its own layout file: [exo_playback_control_view.xml](https://github.com/google/ExoPlayer/blob/release-v2/library/ui/src/main/res/layout/exo_playback_control_view.xml)
- Use of standard ids is required so that child views can be identified, bound to the player and updated in an appropriate way. 
- A full list of the standard ids for each view can be found in the Javadoc for [PlaybackControlView](http://google.github.io/ExoPlayer/doc/reference/index.html?com/google/android/exoplayer2/ui/PlaybackControlView.html) and [SimpleExoPlayerView](http://google.github.io/ExoPlayer/doc/reference/index.html?com/google/android/exoplayer2/ui/SimpleExoPlayerView.html).

### _ExoPlayer Event Listening_

- ExoPlayer.EventListener is a interface that allows you to monitor any changes in the ExoPlayer. It requires that you implement 6 methods but the only one we're interested in is OnPlayerStateChanged.

```java
@Override 
onPlayerStateChanged(boolean playWhenReady, int playbackState)
```
- **playWhenReady** is a play/pause state indicator 
- true = playing, false = paused.
- **playbackState** tells what state the ExoPlayer is in:
	- STATE_IDLE
	- STATE_BUFFERING
	- STATE_READY
	- STATE_ENDED

- Use this method in a if/else statement to log when the ExoPlayer is playing or paused:

```java
    @Override
    public void onPlayerStateChanged(boolean playWhenReady, int playbackState) {
        if((playbackState == ExoPlayer.STATE_READY) && playWhenReady){
            mStateBuilder.setState(PlaybackStateCompat.STATE_PLAYING,
                    mExoPlayer.getCurrentPosition(), 1f);
            Log.d("onPlayerStateChanged:", "PLAYING");
        } else if((playbackState == ExoPlayer.STATE_READY)){
            mStateBuilder.setState(PlaybackStateCompat.STATE_PAUSED,
                    mExoPlayer.getCurrentPosition(), 1f);
            Log.d("onPlayerStateChanged:", "PAUSED");
        }
        mMediaSession.setPlaybackState(mStateBuilder.build());
        showNotification(mStateBuilder.build());
    }
```

- We will use this method to update the MediaSession

### _Add Media Session - Part 1_

#### Creating a [MediaSession](https://developer.android.com/guide/topics/media-apps/working-with-a-media-session.html)

1. Create a MediaSessionCompat object (when the activity is created to initialize the MediaSession)

```java
mMediaSession = new MediaSessionCompat(this, TAG);
```
2. Set the Flags using the features you want to support

```java
mMediaSession.setFlags(
	MediaSessionCompat.FLAG_HANDLES_MEDIA_BUTTONS | 
		MediaSessionCompat.FLAG_HANDLES_TRANSPORT_CONTROLS);
```
3. Set an optional Media Button Receiver component

```java
mMediaSession.setMediaButtonReceiver(null);
```
- We will set it to null since we don't want a MediaPlayer button to start our app if it has been stopped

4. Set the available actions and initial state

```java
mStateBuilder = new PlaybackStateCompat.Builder()
	.setActions(
		PlaybackStateCompat.ACTION_PLAY |
		PlaybackStateCompat.ACTION_PAUSE |
		PlaybackStateCompat.ACTION_PLAY_PAUSE |
		PlaybackStateCompat.ACTION_SKIP_TO_PREVIOUS);

mMediaSession.setPlaybackState(mStateBuilder.build());
```

5. Set the callbacks

```java
mMediaSession.setCallback(new MySessionCallback());
```
6. Start the session

```java
mMediaSession.setActive(true);
```
7. End the session when it is no longer needed (when the activity is destroyed to release the MediaSession)

```java
mMediaSession.setActive(false);
```

```java
/**
* Media Session Callbacks, where all external clients control the player.
*/

private class MySessionCallback extends MediaSessionCompat.Callback {
	@Override
	public void onPlay() {
		mExoPlayer.setPlayWhenReady(true);
	}

	@Override 
	public void onPause() {
		mExoPlayer.setPlayWhenReady(false);
	}

	@Override
	public void onSkipToPrevious(){
		mExoPlayer.seekTo(0);
	}
}
```
- We need to make sure that wether the app is controlled from the internal client (ExoPlayerView) or any external client (through the MediaSession), both the ExoPlayer and the MediaSession contain the State information and they remain in sync.

#### Internal Client

- We need to make sure that when the State changes from the UI, the MediaSession is updated by using this method:

```java
// Call this every time the Player State changes

mStateBuilder.setState(PlaybackStateCompat.STATE_PLAYING, mExoPlayer.getCurrentPosition(), 1f)

mMediaSession.setPlaybackState(mStateBuilder.build());
```

### _Add Media Session - Part 2_

- Now every time you press the Media Button in the ExoPlayerView, our MediaSession will have its state updated

- Next, we will need to make sure that external clients can control the ExoPlayer as well

- Our MediaSession.Callback will automatically be called by external clients

- We just need to make sure it calls the appropriate ExoPlayer methods which will then trigger the ExoPlayer.EventListener and therefore update the MediaSession 

- The only problem is we still don't have any external player setup. Let's create a Media Style Notification aka an external client

### _Add MediaStyle Notification_

- To verify that our MediaSession is working as intended, we will add a [MediaStyle Notification](https://developer.android.com/reference/android/support/v7/app/NotificationCompat.MediaStyle.html)

- Let's create a method called showNotification:

```java
private void showNotification(PlaybackCompat state){
	NotificationCompat.Builder builder = new NotificationCompat.Builder(this);

	int icon;
	String play_pause;
	if(state.getState() == PlaybackCompat.STATE_PLAYING){
		icon = R.drawable.exo_controls_pause;
		play_pause = getString(R.string.pause);
	}
	else {
		icon = R.drawable.exo_controls_play;
		play_pause = getString(R.string.play);
	}

	NotificationCompat.Action playPauseAction = new NotificationCompat.Action(
		icon, play_pause,
		MediaButtonReceiver.buildMediaButtonPendingIntent(this,
			PlaybackStateCompat.ACTION_PLAY_PAUSE));

	NotificationCompat.Action restartAction = new NotificationCompat.Action(
		R.drawable.exo_controls_previous, getString(R.string.restart),
		MediaButtonReceiver.buildMediaButtonPendingIntent(this,
			PlaybackStateCompat.ACTION_SKIP_TO_PREVIOUS));

	PendingIntent contentPendingIntent = PendingIntent.getActivity
		(this, 0, new Intent(this, QuizActivity.class), 0);

	builder.setContentTitle(getString(R.string.guess))
		.setContentText(getString(R.string.notification_text))
		.setContentIntent(contentPendingIntent)
		.setSmallIcon(R.drawable.ic_music_note)
		.setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
		.addAction(restartAction)
		.addAction(playPauseAction)
		.setStyle(new Notification.MediaStyle()
			.setMediaSession(mMediaSession.getSessionToken())
			.setShowActionsInCompatView(0,1));

	mNotificationManager = (NotificationManager) getSystemService(NOTIFICATION_SERVICE);
	mNotificationManager.notify(0, builder.build());
}
```
- We call our new showNotification in the OnPlayerStateChanged method

```java
showNotification(mStateBuilder.build());
```
- We should also call cancelAll on the NotificationManager when the activity is destroyed

```java
private void releasePlayer(){
	mNotificationManager.cancelAll();
	mExoPlayer.stop();
	mExoPlayer.release();
	mExoPlayer = null;
}
```
### _Add Media Button Receiver_

- We need to create a BroadcastReceiver with an intent-filter for the MediaButton Intent Action (AndroidManifest)

```xml
<application 
	...
	<receiver android:name=".QuizActivity$MediaReceiver">
		<intent-filter>
			<action android:name="android.intent.action.MEDIA_BUTTON" />
		</intent-filter>
	</receiver>
</application>
```
- We can create it as a static inner class inside the MainActivity

```java
/**
* Broadcast Receiver registered to receive the MEDIA_BUTTON intent coming from clients
*/

public static class MediaReceiver extends BroadcastReceiver {
	
	public MediaReceiver(){
	}
	
	@Override
	public void onReceive(Context context, Intent intent){
		MediaButtonReceiver.handleIntent(mMediaSession, intent);
	}
}
```
### [_Migrating MediaStyle notifications to support Android O_](https://medium.com/google-developers/migrating-mediastyle-notifications-to-support-android-o-29c7edeca9b7)

#### _Change your import statements_

```
import android.support.v4.app.NotificationCompat;
import android.support.v4.content.ContextCompat;
import android.support.v4.media.app.NotificationCompat.MediaStyle;
```
#### _You might have an import statement from v7, that you no longer need:_

```
import android.support.v7.app.NotificationCompat;
```
#### _In your build.gradle, you now only need to import the media-compat support library._

```
implementation ‘com.android.support:support-media-compat:26.+’
```

### _Use NotificationCompat with channels_
#### The v4 support library now has a new constructor for creating notification builders:

```
NotificationCompat.Builder notificationBuilder =
        new NotificationCompat.Builder(mContext, CHANNEL_ID);
```
#### The NotificationCompat class does not create a channel for you. You still have to create a channel yourself.

```
private static final String CHANNEL_ID = "media_playback_channel";

    @RequiresApi(Build.VERSION_CODES.O)
    private void createChannel() {
        NotificationManager
                mNotificationManager =
                (NotificationManager) mContext
                        .getSystemService(Context.NOTIFICATION_SERVICE);
        // The id of the channel.
        String id = CHANNEL_ID;
        // The user-visible name of the channel.
        CharSequence name = "Media playback";
        // The user-visible description of the channel.
        String description = "Media playback controls";
        int importance = NotificationManager.IMPORTANCE_LOW;
        NotificationChannel mChannel = new NotificationChannel(id, name, importance);
        // Configure the notification channel.
        mChannel.setDescription(description);
        mChannel.setShowBadge(false);
        mChannel.setLockscreenVisibility(Notification.VISIBILITY_PUBLIC);
        mNotificationManager.createNotificationChannel(mChannel);
    }
```

#### _Here’s code that creates a MediaStyle notification with NotificationCompat._
```
// You only need to create the channel on API 26+ devices
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
  createChannel();
}
NotificationCompat.Builder notificationBuilder =
       new NotificationCompat.Builder(mContext, CHANNEL_ID);
notificationBuilder
       .setContentTitle(getString(R.string.guess))
                .setContentText(getString(R.string.notification_text))
                .setContentIntent(contentPendingIntent)
                .setSmallIcon(R.drawable.ic_music_note)
                .setVisibility(NotificationCompat.VISIBILITY_PUBLIC)
                .addAction(restartAction)
                .addAction(playPauseAction)
                .setStyle(new MediaStyle()
                        .setMediaSession(token)
                        .setShowActionsInCompactView(0, 1));
```
### _Android Media Framework Extras_
#### Audio Focus

- This is how the Android framework knows about different applications using audio. If you want your app to fade out when other important notifications (such as navigation) occur, you'll need to learn how your app can "hop in line" to be the one in charge of audio playback, until another app requests focus.

#### Noisy Intent

- Imagine you were listening to a song at full volume, and you trip and yank out the headphones from the audio port. The android framework sends out the [ACTION_ AUDIO_ BECOMING_ NOISY](https://developer.android.com/reference/android/media/AudioManager.html#ACTION_AUDIO_BECOMING_NOISY) intent when this occurs. This allows you to register a broadcast receiver and take a specific action when this occurs (like pausing the music).

#### Audio Stream

- Android uses separate audio streams for playing music, alarms, notifications, the incoming call ringer, system sounds, in-call volume, and DTMF tones. This allows users to control the volume of each stream independently.

- By default, pressing the volume control modifies the volume of the active audio stream. If your app isn't currently playing anything, hitting the volume keys adjusts the ringer volume. To ensure that volume controls adjust the correct stream, you should call [setVolumeControlStream()](https://developer.android.com/reference/android/app/Activity.html#setVolumeControlStream(int)) passing in [AudioManager.STREAM_MUSIC](https://developer.android.com/reference/android/media/AudioManager.html#STREAM_MUSIC).

### _ExoPlayer Extras_

#### Subtitle Side Loading

- Given a video file and a separate subtitle file, MergingMediaSource can be used to merge them into a single source for playback.

```java
MediaSource videoSource = new ExtractorMediaSource(videoUri, ...);
MediaSource subtitleSource = new SingleSampleMediaSource(subtitleUri, ...);
// Plays the video with the sideloaded subtitle.
MergingMediaSource mergedSource =
    new MergingMediaSource(videoSource, subtitleSource);
```
#### Looping a video

- A video can be seamlessly looped using a LoopingMediaSource. The following example loops a video indefinitely. It’s also possible to specify a finite loop count when creating a LoopingMediaSource.

```java
MediaSource source = new ExtractorMediaSource(videoUri, ...);
// Loops the video indefinitely.
LoopingMediaSource loopingSource = new LoopingMediaSource(source);
```
