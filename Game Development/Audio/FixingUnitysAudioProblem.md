# Fixing Unity's Audio System

## The most accurate timer you've ever timed

If you've tried to make rhythm games in Unity, chances are, you'll run into problems using Unity's built-in audio system:

If you use `AudioSource.Time` to determine the time, you may find it hard to extrapolate time to before the song starts (for a countdown before notes spawn), or after the song ends (to give a brief moment of silence before showing the results).

On the other hand, `AudioSettings.dspTime`, although providing an accurate timestamp that isn't confined to the range of audio clips, it only points to the **start** position of the current audio buffer, giving the same value across multiple frames.

Here's a rough depiction of how this works:

![A timeline showing the current buffer time being 2, yet the actual time is 2.42.](.\FixingUnitysAudioProblem\BufferExplanation.png)

To address this issue, we could use **linear regression** to map the accurate but buffered `dspTime` to the less accurate but smooth `Time.realtimeSinceStartup`, and this works surprisingly well!

Check out [Freya Holmér's xeet](https://twitter.com/FreyaHolmer/status/845954862403780609) on X (formerly Twitter) for a different visualization and a comparison between different timing methods.

## The pause-resume problem

In the previous section, we solved an issue with Unity's audio system. However, this isn't the only issue we need to conquer. In fact, when you pause and resume from _some_ .mp3 files, you might notice `dspTime` and the actual music going out of sync, despite the fact that you remember they are indeed in sync at the start of the song.

It turns out, this happens to .mp3 files because they contain [coarse seek tables](https://www.un4seen.com/forum/?topic=14119.msg98297#msg98297).

Here's an excerpt from [hydrogenaudio's wiki page on variable bitrate (VBR) .mp3 files](https://wiki.hydrogenaud.io/index.php?title=Variable_Bitrate):

> VBR formats often include metadata at the beginning of the file to indicate the total duration and provide a "seek table" of the offsets, within the file, of a set number of points, each a fixed duration apart.
>
> For example, the seek table may have 100 offsets to represent 1%, 2%, 3%, etc. of the way toward the total duration. This feature is sometimes also used in CBR formats, so that tags don't confound file-size based estimates of duration and seek points.

As you can see, audio seeking gets way more inaccurate as the seek table size decreases.

![Music time desyncs with dspTime, as it moves to a position described by the seek table, while dspTime is unaffected and resumes from its original position.](.\FixingUnitysAudioProblem\SeekTableExplanation.png)

At the moment, I haven't found a fix within Unity that can be pulled off quite easily, so unfortunately, if you don't want to use any 3rd party libraries, you'll have to use an audio format that don't uses seek tables, like [ogg vorbis](https://xiph.org/vorbis/)!

## Actually fixing it with BASS

But let's face it, if you intend to run a community-based rhythm game, it's natural for players to try to import a VBR .mp3 file. So instead of running away, let's talk about actually fixing this.

[Un4seen.BASS](https://www.un4seen.com/bass.html) is a cross-platform audio library with a wide range of supported audio formats, including mp3, ogg, wav, and so much more. Best of all, it's **free for non-commercial use**!

What we're interested in is their mp3 handling. Here's an interesting observation in their documentations:

> For MP3/MP2/MP1 streams, unless the file is scanned via the `BASS_POS_SCAN` flag or the `BASS_STREAM_PRESCAN` flag at stream creation, seeking will be **approximate** but generally still quite accurate.

So, they provide a way to scan an audio stream before playback to basically eliminate the inaccurate seek positions provided by the seek table. Nice.

## Bringing it into Unity

Let's first grab the appropriate DLLs from the BASS homepage. You can find them on the right of the download label.

![BASS's homepage header, with 5 download links for different platform targets on the right.](.\FixingUnitysAudioProblem\WhereToDownloadBass.png)

Additionally, we'll need BASSmix, an add-on for channel mixing. We'll need this to retrieve time that isn't restrained to the bounds of audio clips, as mentioned above. It also handles time smoothing by itself, which is really cool.

You can find BASSmix a few scrolls down the page.

![The download links for BASSmix on the homepage.](.\FixingUnitysAudioProblem\WhereToDownloadBassMix.png)

Then let's put all its included libraries inside Unity. Create a `Plugin` folder, and two separate folders for the core BASS library, and another for the BASSmix library. Inside each of them, extract the binaries for each platform.

![Three BASS binary folders inside the directory Assets/Plugins/Un4seen.Bass/Core.](.\FixingUnitysAudioProblem\PuttingBassInUnity.png)

Click on each of the binaries, and make sure they're loaded on the correct platforms.

![The inspector window for a 32-bit Windows BASS binary, with the standalone x86 option enabled.](.\FixingUnitysAudioProblem\BassPluginPlatformSettings)

Since BASS is compiled from C, it might be a good idea to grab a wrapper so we don't need to deal with those `DLLImport`s. I'll use [ManagedBASS](https://github.com/ManagedBass/ManagedBass) for this post.

ManagedBASS is available as a NuGet package. To use it in Unity, you can use [NuGetForUnity](https://github.com/GlitchEnzo/NuGetForUnity). Install it into your project, search for ManagedBASS, and install their wrappers for both BASS and BASSmix.

![ManagedBASS wrappers in NuGetForUnity, with its BASS wrapper named ManagedBass, and its BASSmix wrapper named ManagedBass.Mix](.\FixingUnitysAudioProblem\WhereToDownloadBassWrappers.png)

Optionally, if you're going all in with BASS, you can disable Unity's audio system under _Edit -> Project Settings -> Project -> Audio_.

![The project settings window, with the Disable Unity Audio option highlighted](.\FixingUnitysAudioProblem\DisableUnityAudio)

That's it for the setup, let's move on to the implementations!

## Putting BASS into Unity

To be able to use BASS, we'll need to initialize it. Let's create a regular c# class called `AudioSystem` to do just that:

```csharp
using ManagedBass;
using UnityEngine;

namespace Audio
{
    public static class AudioSystem
    {
        public static bool Initialized { get; private set; }

        [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSplashScreen)]
        public static void Initialize()
        {
            Bass.Init();
            Initialized = true;

            Application.quitting     += Dispose;
            Application.focusChanged += OnApplicationFocusChanged;

            Bass.UpdatePeriod         = 5;
            Bass.DeviceBufferLength   = 10;
            Bass.PlaybackBufferLength = 100;
            Bass.DeviceNonStop        = true;

            // without this, if bass falls back to direct-sound legacy mode the audio playback offset will be way off.
            Bass.Configure(Configuration.TruePlayPosition, 0);

            // For iOS devices, set the default audio policy to one that obeys the mute switch.
            Bass.Configure(Configuration.IOSMixAudio, 5);

            // Always provide a default device. This should be a no-op, but we have asserts for this behaviour.
            Bass.Configure(Configuration.IncludeDefaultDevice, true);

            // Enable custom BASS_CONFIG_MP3_OLDGAPS flag for backwards compatibility.
            Bass.Configure((Configuration)68, 1);

            // Disable BASS_CONFIG_DEV_TIMEOUT flag to keep BASS audio output from pausing on device processing timeout.
            // See https://www.un4seen.com/forum/?topic=19601 for more information.
            Bass.Configure((Configuration)70, false);
        }

        private static void OnApplicationFocusChanged(bool focused)
        {
            if (!Application.isEditor)
                Bass.GlobalStreamVolume = Bass.GlobalSampleVolume = focused ? 10_000 : 0;
        }

        private static void Dispose()
        {
            Initialized = false;
            Bass.Free();

            Application.focusChanged -= OnApplicationFocusChanged;
            Application.quitting     -= Dispose;
        }
    }
}
```

Some of the initialization code is directly pulled from [osu!lazer's implementation](https://github.com/ppy/osu-framework/blob/fdb335b45ed9f983ab4a0d1da620f3e10bf8e586/osu.Framework/Audio/AudioManager.cs#L407), give their code a good look if you're interested!

To be safe, I made the initialization code run before the splash screen shows, so all subsequent audio requests from different classes are guaranteed an initialized audio system. Generally, if you initialize BASS before any request gets a chance to run, you should be fine.

Next, let's import our audio clips. ManagedBASS provides multiple methods of loading in music: from a file path, from a url, from a byte array, etc.

For example, you can do the following to import audio from a file:

```csharp
Bass.CreateStream(fullFilePath, 0, 0, BassFlags.Default)
```

`Bass.CreateStream` returns an integer representing a channel handle. You can then use this integer to ask BASS about its info, seek its position, stop it, etc.

Since we want to make VBR mp3 seeking accurate, let's also include a `Prescan`, or a `Scan` flag:

```csharp
Bass.CreateStream(fullFilePath, 0, 0, BassFlags.Prescan | flags)
```

I've opted to use `Prescan` instead of `Scan` here. When I first implemented it, I always thought they're interchangeable, so I chose an option that _seemed_ more safe.

In more serious terms though, the official documentation mentioned this:

> Chained OGG files containing multiple logical bitstreams are supported, but they will need to be scanned to get their length or to seek in them. That scanning will be done at stream creation or at the first [BASS_ChannelGetLength](https://www.un4seen.com/doc/bass/BASS_ChannelGetLength.html) or [BASS_ChannelSetPosition](https://www.un4seen.com/doc/bass/BASS_ChannelSetPosition.html) call, depending on whether the [BASS_CONFIG_OGG_PRESCAN](https://www.un4seen.com/doc/bass/BASS_CONFIG_OGG_PRESCAN.html) option is enabled (or the BASS_STREAM_PRESCAN flag is used).

Basically, using `Prescan` allows us to also support chained ogg files. Although we might only need this feature under niche applications (since we can manually chain music files together).

You may also find loading from bytes useful. Since we no longer have Unity's native audio clips, we'll need to handle loading from in-project audio files ourselves.

To load audio files as binary files directly, you'll have to change their file extension to `.bytes`. Otherwise, Unity still regards them as audio files, and reading them gives you exceptions. You can try it out by putting the audio files inside the resources folder, and loading them with `Resources.Load<TextAsset>(pathToAudio).bytes`.

Additionally, we can't really use `Bass.CreateStream` directly on bytes. Internally, ManagedBASS adds a channel sync callback that gets triggered when the audio should be freed. This should be fine by itself, but Unity, when running under IL2CPP, **doesn't support instanced native callbacks** (As of Unity 2022). So we'll have to dig into the source code of ManagedBASS, reconstruct the loading-from-bytes procedure, and exclude the part where they add the channel sync:

```csharp
var gcPin  = GCHandle.Alloc(audioData, GCHandleType.Pinned);
var handle = Bass.CreateStream(gcPin.AddrOfPinnedObject(), 0, audioData.Length, flags);
if (handle == 0)
{
    gcPin.Free();
    return null;
}

return handle;
```

We can then:

- Use `Bass.ChannelPlay(channelHandle)` to play the music
- Use `Bass.ChannelBytes2Seconds(channelHandle, Bass.ChannelGetPosition(channelHandle))` to retrieve its time
- Use `Bass.ChannelSetPosition(channelHandle, Bass.ChannelSeconds2Bytes(channelHandle, time))` to seek the audio to a position
- etc.

Yay!

At this point, we can play the music, but the time information we receive are clamped. So, let's put them in mixer channels.

Mixer channels are special channels introduced by BASSmix. Because they only accept decode channels as inputs, we have to make our source audio streams sit on decode channels. Let's modify our stream creation code to include the `Decode` flag:

```csharp
Bass.CreateStream(fullFilePath, 0, 0, BassFlags.Decode | BassFlags.Prescan | flags)
```

By marking a channel as a decode channel, it can no longer be paused. To help with this, we can fake a pause by muting it instead. We can still use its seeking function the same way we do it on a normal channel though.

To create a mixer channel, you can use `BassMix.CreateMixerStream`. To make the mixer channel provide a non-stop timer, we'll need to put in a `MixerNonStop` flag. For example, if you want a mixer with a dual channel 44100Hz output, you can do the following:

```csharp
BassMix.CreateMixerStream(44100, 2, BassFlags.MixerNonStop | BassFlags.MixerChanNoRampin)
```

I also included a `MixerChanNoRampIn` flag to "avoid ramping-in at the start". This flag is pretty commonly added onto mixer channels, but I wasn't able to find more information on why it's a good practice. If you care, do try removing this flag and see what happens.

Finally, we can then add a decoding audio channel to the mixer channel with `BassMix.MixerAddChannel(mixerChannelHandle, decodingChannelHandle, BassFlags.Default)`. You should now be able to hear music output from the mixer channel.

## Managing time

Alright! We've successfully implemented the necessary parts to get audio through BASS. If you tried to retrieve time information from a mixer channel through `Bass.ChannelBytes2Seconds`, you might see values very similar to those from `AudioSettings.dspTime`, except in the BASS case the value is actually smooth and increases every frame.

Which means, we'll need a system to manage the time we get from BASS (we'll call this an internal time), and convert it to a value relative to the audio clip's time, with 0 being the start of the song and where negative values mean the song hasn't started, as well as a way to pause, resume, and seek time.

First of all, let's create a MonoBehaviour class called `MusicManager`, and a method to load the music:

```csharp
public  double  StartTime { get; set; }
private double  MixerTime => Bass.ChannelBytes2Seconds(mixerChannelHandle, Bass.ChannelGetPosition(mixerChannelHandle));

// 3-second countdown before the music kicks in
private const double ScheduleDuration = 3;

private void ScheduleMusic(int streamChannelHandle)
{
    StartTime = MixerTime + ScheduleDuration;
}
```

We then update the current time in the `Update` method. Of course, we want to skip updating altogether if the game is paused too:

```csharp
public double Time { get; set; }
public bool   Paused { get; set; }

private void Update()
{
    if (Paused)
        return;

    var currentTime = MixerTime - StartTime

    if (currentTime >= 0)
    {
        // When the music kicks in (currentTime >= 0) but
        // the previous frame it didn't (_started == false),
        // We move the audio position to the current time, and enable the volume
        if (!_started)
        {
            _started = true;
	        Bass.ChannelSetPosition(streamChannelHandle, Bass.ChannelSeconds2Bytes(streamChannelHandle, currentTime));
    	    Bass.ChannelSetAttribute(streamChannelHandle, ChannelAttribute.Volume, 1);
        }
    }
    else
    {
        // If the music didn't begin, but
        // on the previous frame the music played (Seek() was called manually to a negative value),
        // Fake a pause by muting the audio, and set the audio position to the start
        if (_started)
        {
            _started     = false;
	        Bass.ChannelSetAttribute(streamChannelHandle, ChannelAttribute.Volume, 0);
            Bass.ChannelSetPosition(streamChannelHandle, Bass.ChannelSeconds2Bytes(streamChannelHandle, 0));
        }
    }
}
```

As for `Seek`, we just need to change `StartTime` so that `MixerTime - StartTime` is equal to the target seek time (i.e. the time it took from the start to the seeking position):

```csharp
public void Seek(double time)
{
    time = Math.Clamp(time, 0, Bass.ChannelBytes2Seconds(streamChannelHandle, Bass.ChannelGetLength(streamChannelHandle)));
    Bass.ChannelSetPosition(streamChannelHandle, Bass.ChannelSeconds2Bytes(streamChannelHandle, time));
    StartTime = MixerTime - time;
}
```

Finally, pausing and resuming come in a pair. Since the decoding channel and the mixer channel never stops ticking, we need to store the time when pause was requested, and seek to that time when we resume:

```csharp
private double _timeBeforePause;

public void Pause()
{
    if (_started)
        Bass.ChannelSetAttribute(streamChannelHandle, ChannelAttribute.Volume, 0);
    _timeBeforePause = currentTime;
    Paused           = true;
}

public void Resume()
{
    Seek(_timeBeforePause - 1);
    if (_started)
    {
        Bass.ChannelSetAttribute(streamChannelHandle, ChannelAttribute.Volume, 1);
    }
    Paused = false;
}
```

## Conclusion

And we're done! A fully functional audio system in Unity, with an accurate yet smooth clock, and support for a variety of audio formats with precise seeking.

Of course, on top of managing pieces of music, BASS can also help manage short oneshots. You can find more info on the BASS and BASS.NET documentation, and build on top of them to meet the requirements of actual games.

I hope you find this post useful, good luck out there <3
