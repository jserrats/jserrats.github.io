---
layout: post
title: Reversing a simple Unity Android game
subtitle: Understanding how Unity games are structured and an introduction to reversing a Dll file
thumbnail-img: /assets/img/reversing-unity/unity-logo.png
share-img: /assets/img/reversing-unity/unity-logo.png
tags: [reversing, android]#
---

## About the game

This is a simple game (that I won't be naming just in case, I don't want to get in any trouble) that consists in popping balloons. Each balloon popped adds into a currency that can be used to unlock stuff (more balloons, wallpapers and stuff like that). As seen in the video, the balloons appear very slowly, so we'll find a quicker way to obtain currency.

<iframe width="560" height="315" src="https://www.youtube.com/embed/IDRWgid3mS0" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Shared Preferences

Before doing any work on an app it is usually worth it to check which information is stored on the Shared Preferences. From the [Android documentation](https://developer.android.com/training/data-storage/shared-preferences):

> A SharedPreferences object points to a file containing key-value pairs and provides simple methods to read and write them. Each SharedPreferences file is managed by the framework and can be private or shared.

Ultimately it is a simple clear text XML file stored in the Android File system, that is easily accessible using `adb`.

```
OnePlus5T:/data/data/com.turner.jerrysgame/shared_prefs # ls -la
total 40
drwxrwx--x 2 u0_a226 u0_a226 4096 2021-01-04 10:39 .
drwx------ 5 u0_a226 u0_a226 4096 2021-01-04 10:35 ..
-rw-rw---- 1 u0_a226 u0_a226 1694 2021-01-04 10:39 APP_MEASUREMENT_CACHE.xml
-rw-rw---- 1 u0_a226 u0_a226 2284 2021-01-04 10:39 com.turner.jerrysgame.v2.playerprefs.xml
-rw-rw---- 1 u0_a226 u0_a226  354 2021-01-04 10:35 embryo.xml
```

Here is a dump of the Shared preferences file of this game:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <int name="GAME_SAVE_DATA_balloonCurrencySpentKey" value="0" />
    <int name="Screenmanager%20Resolution%20Height" value="2160" />
    <string name="unity.player_sessionid">XXXXXXXXXXXXXXXXXXXX</string>
    <int name="GAME_SAVE_DATA_numGamesPlayedKey" value="3" />
    <float name="GAME_SAVE_DATA_midSaveGameTimeKey" value="0.0" />
    <string name="GAME_SAVE_DATA_gameItemPopCountKey">ARUAAAASAAAACgAAABIAAAAS
        AAAAAAAAAAEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
        AAAAAAAAAAAAAAAAQAAAA%3D%3D</string>
    <int name="Screenmanager%20Resolution%20Width" value="1080" />
    <int name="GAME_SAVE_DATA_numAppSessionsKey" value="2" />
    <int name="GAME_SAVE_DATA_mainMenuPlaySelectedKey" value="0" />
    <int name="GAME_SAVE_DATA_storeIntroDisplayedKey" value="1" />
    <int name="GAME_SAVE_DATA_midSaveGameStateKey" value="0" />
    <int name="GAME_SAVE_DATA_scoreYetToBeSubmittedKey" value="43" />
    <int name="GAME_SAVE_DATA_midSaveScoreAchievementIndex" value="0" />
    <int name="GAME_SAVE_DATA_balloonCurrencyKey" value="51" />
    <int name="GAME_SAVE_DATA_didSaveMidGameKey" value="0" />
    <int name="GAME_SAVE_DATA_totalBalloonsEarnedKey" value="56" />
    <string name="GAME_SAVE_DATA_unlockedStoreItemsKey">AhYAAAAAAAE%3D</string>
    <int name="__UNITY_PLAYERPREFS_VERSION__" value="1" />
    <int name="GAME_SAVE_DATA_midSaveBalloonsPoppedKey" value="0" />
    <int name="GAME_SAVE_DATA_firstRunCompleteKey" value="1" />
    <int name="GAME_SAVE_DATA_boostsFeatureUnlockedKey" value="1" />
    <int name="Screenmanager%20Fullscreen%20mode" value="-1" />
    <int name="GAME_SAVE_DATA_unlockAllPurchasedKey" value="0" />
    <int name="GAME_SAVE_DATA_boostIntroDisplayedKey" value="0" />
    <string name="GAME_SAVE_DATA_loadedBoostsKey">AhYAAAAAAAA%3D</string>
    <int name="GAME_SAVE_DATA_isSoundtrackEnabledKey" value="1" />
    <string name="GAME_SAVE_DATA_equippedStoreItemsKey">AhYAAAAAAAA%3D</string>
    <int name="GAME_SAVE_DATA_isSoundFXEnabledKey" value="1" />
    <float name="GAME_SAVE_DATA_midSaveTimeLeftKey" value="0.0" />
    <string name="GAME_SAVE_DATA_startingBoostsKey">AhYAAAAAAAA%3D</string>
    <int name="GAME_SAVE_DATA_saveDataVersionNumberKey" value="0" />
</map>
```

As we can see there are several entries that seem interesting for our purposes, specifically

```xml
<int name="GAME_SAVE_DATA_balloonCurrencyKey" value="51" />
<int name="GAME_SAVE_DATA_totalBalloonsEarnedKey" value="56" />
```

The are values that seem to contain the in-game items that we have unlocked stored using a base64 encoded byte array, but we'll look into this another day.

Let's modify this value, relaunch the app and see what happens

```xml
<int name="GAME_SAVE_DATA_balloonCurrencyKey" value="995" />
<int name="GAME_SAVE_DATA_totalBalloonsEarnedKey" value="1004" />
```

![](/assets/img/reversing-unity/screenshot.png)

> Note how the in-game menu has an in-app purchase option to get 100k ballons for some real world money

Mission successful! But this was a quite easy hack and not really a challenge... We'll try to modify the game to obtain 9999 points for each balloon popped instead of one.

## Reversing Unity Games

As most Android Games out there this game is programmed using the Unity Game Engine. This means that when reversing the game we won't find the game logic in `smali` files since it is programed inside Unity and not java/kotlin.

As we can see when decompiling, there are very few java classes, and it seems mostly boilerplate code.

![](/assets/img/reversing-unity/jadx1.png)

### Extracting the DLL from the APK file

To extract all the contents of the apk file we can use `apktool`

```
jserrats@glados:~/reversing-android/com.turner.jerrysgame$ apktool d  base.apk 
I: Using Apktool 2.4.0-dirty on base.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/jserrats/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

Unity stores the logic in `dll` files stored under `/base/assets/bin/Data/Managed`. In particular, we are interested in `Assembly-CSharp.dll`

```
jserrats@glados:~/reversing-android/com.turner.jerrysgame/base/assets/bin/Data/Managed$ lsl                                                 
total 17M                                                                                                         
-rw-rw-r-- 1 jserrats jserrats  591K Jan  4 11:28 Assembly-CSharp.dll 
```

### Modifying the DLL file

In order to work with the DLL file I'll be using [dnSpy](https://github.com/dnSpy/dnSpy), which it is an awesome tool (but only works on windows :/ ). Also keep in mind that you'll need to open all the DLL files in the project, or we won't be able to recompile the DLL since there will be missing dependencies.

We can see that inside `Assembly-CSharp.dll` there are all the classes that contain the game logic.

![](/assets/img/reversing-unity/dnspy.png)

After browsing for some time all the classes, we seem to find the class that adds the score for each balloon popped.

![](/assets/img/reversing-unity/dnspy2.png)

Doing right click and selecting `Edit C# Method` we can open a window where we can edit code. Doing this we are decompiling the DLL into C#, modifying the logic and then recompiling again into a DLL. Another option is to directly edit the DLL at [CLI level](https://en.wikipedia.org/wiki/Common_Language_Infrastructure).

![](/assets/img/reversing-unity/dnspy3.png)

After modifying the line and hitting `Compile` we get several compilation errors.. It seems that the debugging attributes are not valid here. Rather than trying to fix this, I have opted for the quickest way, which is to remove all the conflictive debugging attributes (we are not debugging this application anyway) and proceed.

![](/assets/img/reversing-unity/dnspy4.png)

Then we can save the DLL using the option `Save Module`

![](/assets/img/reversing-unity/dnspy5.png)

### Repackaging the APK

With the modified DLL in our hands, we can proceed into repackaging the APK so it can be installed. To do so we place the modified DLL into the original folder generated by Apktool.

```
jserrats@glados:~/reversing-android/com.turner.jerrysgame/base2$ cp ~/vmshared/Assembly-CSharp.dll assets/bin/Data/Managed/Assembly-CSharp.dll
```

Then we can repackage it again using the `build` option:

```
jserrats@glados:~/reversing-android/com.turner.jerrysgame$ apktool b base2
I: Using Apktool 2.4.0-dirty                        
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
W: aapt: brut.common.BrutException: brut.common.BrutException: Could not extract resource: /prebuilt/linux/aapt_64 (defaulting to $PATH binary)
I: Copying libs... (/lib)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk...
```

Keep in mind that when building the application the original signature from the developer is no longer valid (that is why it exists in the first place), and to install it into our device we'll have to sign it. To do so whe can use our own self signed key.

```
jserrats@glados:~/reversing-android$ jarsigner -verbose -sigalg SHA1withRSA -digestalg SHA1 -storepass 123456 -keystore android.keystore com.turner.jerrysgame/base2/dist/base.apk android jar signed.

Warning: 
The signer's certificate is self-signed.
The SHA1 algorithm specified for the -digestalg option is considered a security risk. This algorithm will be disabled in a future update.
The SHA1withRSA algorithm specified for the -sigalg option is considered a security risk. This algorithm will be disabled in a future update.
```

The only step left is to install the apk again using `adb`. Remember to delete the original one first, since Android won't allow the installation of a package with the same name as one already installed with a different signature.

```
jserrats@glados:~/reversing-android$ adb install com.turner.jerrysgame/base2/dist/base.apk 
```

We can now beat get thousands of balloons easily in seconds.

<iframe width="560" height="315" src="https://www.youtube.com/embed/IvyOv8v_WWs" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Conclusions

When I started this experiment I was not expecting to dig into a DLL file while doing an Android reversing! It was really interesting to learn a bit about how Unity integrates with all the platforms and the possibilities of Android development. The security conclusions are obvious:

1. Do not store sensitive information on SharedPreferences
2. Apply some kind of obfuscation on your core game logic

This is specially important if your revenue model is based in an in-app purchase that is useless now that we have unlimited currency!
