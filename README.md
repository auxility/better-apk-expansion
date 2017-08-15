# Updated client library for the Google Play APK Expansion files.

A community maintained version of the Android APK Expansion library, which 
was first published by Google.

## New API

### Overview
We believe that the original APK Expansion Library was slightly over-engineered. 
We propose a new simplified API for the Downloader Library.

The new API does not require the user to implement any additional services or broadcast receivers.
It is still based on the original `DownloaderService`, however the communication with the
service made more transparent.

* Service is started directly through the static methods.
* The service now signals download events through the local 
broadcast, and does not require service binding for the most use cases. To receive
download events user just have to extend `BroadcastDownloaderClient`. 
* User does not have to implement an alarm receiver for signals from the service watch dogs.


### Gradle installation 

In your module's `build.gradle` dependencies:
```GROOVY
compile 'com.github.bolein:better-apk-expansion:5.0.0'
```

In your root project's `build.gradle`:
```GROOVY
allprojects {
// ...
    repositories {
        // ...
        maven { url "https://jitpack.io" } // add this line
    }
}
```

### Usage 
#### Starting the download

The main activity in your application (the one started by your launcher icon) is responsible for verifying whether the expansion files are already on the device and initiating the download if they are not.

Starting the download using the Downloader Library requires the following procedures:

1. Check whether the files have been downloaded.

   The Downloader Library includes some APIs in the Helper class to help with this process:
   * `getExpansionAPKFileName(Context, c, boolean mainFile, int versionCode)`
   * `doesFileExist(Context c, String fileName, long fileSize)`

2. Start the download by calling the static method `DownloaderService.startDownloadServiceIfRequired(Context context, PendingIntent pendingIntent, byte[] salt, String publicKey)`.

    The method takes the following parameters:
    * `context` - Your application's Context.
    * `notificationClient` - A PendingIntent to start your main activity. This is used in the Notification that the DownloaderService creates to show the download progress. When the user selects the notification, the system invokes the PendingIntent you supply here and should open the activity that shows the download progress (usually the same activity that started the download).
    * `salt` - An array of random bytes that the licensing Policy uses to create an Obfuscator. The salt ensures that your obfuscated `SharedPreferences` file in which your licensing data is saved will be unique and non-discoverable.
    * `publicKey` - A string that is the Base64-encoded RSA public key for your publisher account, available from the profile page on the Play Console (see [Setting Up for Licensing](https://developer.android.com/google/play/licensing/setting-up.html)).

    The method returns an integer that indicates whether or not the download is required. Possible values are:
    
    `NO_DOWNLOAD_REQUIRED`: Returned if the files already exist or a download is already in progress.
    `LVL_CHECK_REQUIRED`: Returned if a license verification is required in order to acquire the expansion file URLs.
    `DOWNLOAD_REQUIRED`: Returned if the expansion file URLs are already known, but have not been downloaded.
    
    The behavior for `LVL_CHECK_REQUIRED` and `DOWNLOAD_REQUIRED` are essentially the same and you normally don't need to be concerned about them. In your main activity that calls `startDownloadServiceIfRequired()`, you can simply check whether or not the response is `NO_DOWNLOAD_REQUIRED`. If the response is anything other than `NO_DOWNLOAD_REQUIRED`, the Downloader Library begins the download and you should update your activity UI to display the download progress (see the next step). If the response is `NO_DOWNLOAD_REQUIRED`, then the files are available and your application can start.
    
    Notice that the method may return `NO_DOWNLOAD_REQUIRED` if your APK has no expansion files associated with it (see [Linking OBB files](#linking-obb-files)).
    
    For example:
    ```JAVA
    // You must use the public key belonging to your publisher account
    public static final String BASE64_PUBLIC_KEY = "YourLVLKey";   
    // You should also modify this salt. For security reasons, it has
    // to be truly random.
    public static final byte[] SALT = new byte[] { 1, 42, -12, -1, 54, 98,
             -100, -12, 43, 2, -8, -4, 9, 5, -106, -107, -33, 45, -1, 84
    };
 
    @Override
    public void onCreate(Bundle savedInstanceState) {
        // Check if expansion files are available before going any further
        if (!expansionFilesDelivered()) {
            // Build an Intent to start this activity from the Notification
            Intent notifierIntent = new Intent(this, MainActivity.getClass());
            notifierIntent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK |
                                    Intent.FLAG_ACTIVITY_CLEAR_TOP);
            ...
            PendingIntent pendingIntent = PendingIntent.getActivity(this, 0,
                    notifierIntent, PendingIntent.FLAG_UPDATE_CURRENT);
    
            // Start the download service (if required)
            int startResult =
                DownloaderService.startDownloadServiceIfRequired(this,
                            pendingIntent, SALT, BASE64_PUBLIC_KEY);
            // If download has started, initialize this activity to show
            // download progress
            if (startResult != DownloaderClientMarshaller.NO_DOWNLOAD_REQUIRED) {
                // This is where you do set up to display the download
                // progress (next step)
                ...
                return;
            } // If the download wasn't necessary, fall through to start the app
        }
        startApp(); // Expansion files are available, start the app
    }
   ```
#### Receiving download progress
To receive updates regarding the download progress, you must extend and register the `BroadcastDownloaderClient`. It is
important to use the `register()` and `unregister()` methods provided by the client. Note that receiver will only receive
progress updates if it is registered in the same process as the `DownloaderService` started.

```JAVA
public class SampleDownloaderActivity extends AppCompatActivity {
    private final DownloaderClient mClient = new DownloaderClient(this);
    
    // ...
    
    @Override 
    protected void onStart() {
        super.onStart();
        mClient.register(this);
    }

    @Override 
    protected void onStop() {
        mClient.unregister(this);
        super.onStop();
    }
    
    // ...
    
    class DownloaderClient extends BroadcastDownloaderClient {
    
        @Override 
        public void onDownloadStateChanged(int newState) {
            if (newState == STATE_COMPLETED) {
                // downloaded successfully...
            } else if (newState >= 15) {
                // failed
                int message = Helpers.getDownloaderStringResourceIDFromState(newState);
                Toast.makeText(this, message, Toast.LENGTH_SHORT).show();
            } 
        }
        
        @Override 
        public void onDownloadProgress(DownloadProgressInfo progress) {
            if (progress.mOverallTotal > 0) {
                // receive the download progress
                // you can then display the progress in your activity
                String progress = Helpers.getDownloadProgressPercent(progress.mOverallProgress, 
                    progress.mOverallTotal);
                Log.i("SampleDownloaderActivity", "downloading progress: " + progress);
            }
        }
    }

}
```

#### Communicating back to the service
With the `IDownloaderService` interface, you can send commands to the downloader service, such as to pause and resume the download (`requestPauseDownload()` and `requestContinueDownload()`).
To access an instance of the `IDownloaderService`, you can use the `DownloaderProxy` class.

To use the proxy, you should first establish a binding connection with the service by calling `connect()` method. This
method will bind the proxy to the service and send further commands through the `Messenger`.

For example:

```JAVA
// in your activity class

private final DownloaderProxy mDownloaderProxy = new DownloaderProxy(this);

@Override 
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // ...
    
    // Establish a connection to the service
    mDownloaderProxy.connect();
    
    // You can now use the proxy to issue commands to the 
    // DownloaderService such as `requestPauseDownload()`
    // and `requestContinueDownload()`
    
    // Request current download status for the downloader client
    mDownloaderProxy.requestDownloadStatus();
}

@Override 
protected void onDestroy() {
    super.onDestroy();
    // Don't forget to unbind the proxy when you don't 
    // need it anymore
    mDownloaderProxy.disconnect();
}
```

#### Linking OBB files
In order to test the download, your application must be published in Google Play (alpha or beta release works). Before starting to test the download, make sure that you have associated at least one OBB file with your current application's `versionCode`. Versions that have no associated expansion files can't start the download. You can associate the OBB files when you upload the apk file to the Play Console. 

Check the [APK Expansion Files documentation](https://developer.android.com/google/play/expansion-files.html#Overview) for more information about OBB files.

#### Configuring Play Licensing
The first step in the download process is to obtain a valid licensing response. Make sure that you have correctly set your publisher key. For more info about setting up publisher account check the [documentation](https://developer.android.com/google/play/licensing/setting-up.html#account).

Note that you can also set up your testing environment to always emit a valid licensing response (see the [licensing documentation](https://developer.android.com/google/play/licensing/setting-up.html#test-env) for more info).


## Old API

You may also use the old API, which is maintained only in bug-fix mode. For complete docs on the old Downloader Library refer to [APK Expansion Files](https://developer.android.com/google/play/expansion-files.html).

### Gradle installation 

You must compile the 4.X.X version of the library for the old API.

In your module's `build.gradle` dependencies:
```GROOVY
compile 'com.github.bolein:better-apk-expansion:4.0.1'
```

In your root project's `build.gradle`:
```GROOVY
allprojects {
// ...
    repositories {
        // ...
        maven { url "https://jitpack.io" } // add this line
    }
}
```

### Usage

The complete documentation on the old Downloader Library is available at the Android Developers [website](https://developer.android.com/google/play/expansion-files.html#Overview). 

## Changelog
Version 5.0.0
* Introduced updated API
* Updated documentation

Version 4.0.1
* Fixed notification issues
* Bug fixes

Version 4
* Updated for Marshmallow
   - No longer uses Apache HTTP
   - No longer relies on removed Notification methods
* Changed to refer to Google Play
* Notifications now rely on support library.

Version 3
* Directory structure corrected in distribution. No code changes.

Version 2
* Patch file now gets downloaded.
* Honeycomb devices now supported with ICS-like notifications
* CRC check (from sample) now supports compressed Zip files
* Use of reflection removed to allow easy obfuscation
* Service leak fixed
* Unprintable character removed from ZipResourceFile
* Minor formatting changes
* Additional comments and edits to this file

## Packages

### downloader_library
A library that works with the google_market_licensing library to download APK Expansion files in a background service.  This library depends on the licensing library, and must be built as a library project.
### zip_file 
A library that uses multiple zip files as a virtual read-only filesystem with hooks to support APK Expansion files. This also must be built as an Android library project.
### downloader_sample
A sample application that assumes that zip format files have been uploaded as the main/patch file(s) on Android Market.  It downloads these files and then validates that the CRC's for every entry in the zip match.  This application depends on the downloader_library and the zip_file library. Because of dependency issues involving multiple libraries, you may have to do a clean build after creating each project.

## IMPORTANT THINGS TO KNOW

1) Do not plan to extract the contents of an APK Expansion file.  They are intended to be used in-place.  By not compressing audio and video files and storing them in a Zip file they can be played from within an expansion file.
2) See com.google.android.vending.expansion.downloader/Constants.java to turn on verbose logging in the library.  Please turn it off when you publish
3) You must add your SALT and Public key to the SampleDownloaderService and update the xAPKS structure in SampleDownloaderActivity in order to test with the sample code
4) There is no strong need to include the validator with your Android applications.  It is included as a demonstration, and can be used if you wish.

For more information, see the documentation at http://developer.android.com/guide/market/expansion-files.html

This library depends on the [Android License Verification Library](https://github.com/bolein/better-licensing).

See the licensing documentation at http://developer.android.com/guide/publishing/licensing.html
