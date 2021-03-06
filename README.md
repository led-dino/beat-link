# beat-link

A Java library for synchronizing with beats from Pioneer DJ Link
equipment, and finding out details about the tracks that are playing.

See
[beat-link-trigger](https://github.com/brunchboy/beat-link-trigger#beat-link-trigger) and
[beat-carabiner](https://github.com/brunchboy/beat-carabiner)
for examples of what you can do with this.

[![License](https://img.shields.io/badge/License-Eclipse%20Public%20License%201.0-blue.svg)](#license)

> :construction: beat-link is being worked on very heavily at the
> moment to incorporate some
> [amazing new discoveries](https://github.com/brunchboy/dysentery#robust-metadata-understanding).
> Until that settles down, there may be rapid releases and big changes
> between releases.
>
> :warning: Because beat-link is grew so much for release 0.3.0, it
> was time to split some classes apart, and even split it into multiple
> packages, so I also took the opportunity to make some basic changes to
> the API to position it for future growth and better reliability. This
> means version 0.3.0 is not API compatible with prior
> releases, and any code which compiled against an old release will
> require some rewriting.
>
> :octocat: Since, as far as I know, I am still the only consumer of
> this API, this seemed like a good time to make these breaking
> changes.

## Installing

Beat-link is available through Maven Central, so to use it in your
Maven project, all you need is to include the appropriate dependency.

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/org.deepsymmetry/beat-link/badge.svg)](https://maven-badges.herokuapp.com/maven-central/org.deepsymmetry/beat-link)

Click the **maven central** badge above to view the repository entry
for beat-link. The proper format for including the latest release as a
dependency in a variety of tools, including Leiningen if you are using
beat-link from Clojure, can be found in the **Dependency Information**
section.

Beat link uses [slf4j](http://www.slf4j.org/manual.html) to allow you
to integrate it with whatever Java logging framework your project is
using, so you will need to include the appropriate slf4j binding on
your class path.

If you want to manually install beat-link, you can download the
library from the
[releases](https://github.com/brunchboy/beat-link/releases) page and
put it on your project&rsquo;s class path, along with the
[`slf4j-api.jar`](http://www.slf4j.org/download.html) and the slf4j
binding to the logging framework you would like to use.

You will also need
[ConcurrentLinkedHashmap](https://github.com/ben-manes/concurrentlinkedhashmap)
for maintaining album art caches, so Maven is your easiest bet.

## Usage

See the [API Documentation](http://deepsymmetry.org/beatlink/apidocs/)
for full details, but here is a nutshell guide:

### Finding Devices

The [`DeviceFinder`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/DeviceFinder.html)
class allows you to find DJ Link devices on your network. To activate it:

```java
import org.deepsymmetry.beatlink.DeviceFinder;

// ...

  DeviceFinder.getInstance().start();
```

After a second, it should have heard from all the devices, and you can
obtain the list of them by calling:

```java
  DeviceFinder.getInstance.getCurrentDevices();
```

This returns a list of [`DeviceAnnouncement`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/DeviceAnnouncement.html)
objects describing the devices that were heard from. To find out
immediately when a new device is noticed, or when an existing device
disappears, you can use
[`DeviceFinder.getInstance().addDeviceAnnouncementListener()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/DeviceFinder.html#addDeviceAnnouncementListener-org.deepsymmetry.beatlink.DeviceAnnouncementListener-).

### Responding to Beats

The [`BeatFinder`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/BeatFinder.html)
class can notify you whenever any DJ Link devices on your network report
the start of a new beat:

```java
import org.deepsymmetry.beatlink.BeatFinder;

// ...

  BeatFinder.getInstance().addBeatListener(new BeatListener() {
      @Override
      public void newBeat(Beat beat) {
         // Your code here...
      }
  });

  BeatFinder.getInstance().start();
```

The [`Beat`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html)
object you receive will contain useful information about the state of the
device (such as tempo) at the time of the beat.

> To fully understand how to respond to the beats, you will want to start
> [`VirtualCdj`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html)
> as described in the next section, so it can tell you important
> details about the states of all the players, such as which one is
> the current tempo master. Once that is running, you can pass the `Beat`
> object to [`VirtualCdj.getLatestStatusFor(beat)`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#getLatestStatusFor-org.deepsymmetry.beatlink.DeviceUpdate-)
> and get back the most recent status update received from the device
> reporting the beat.

With just the `Beat` object you can call
[`getBpm()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html#getBpm--)
to learn the track BPM when the beat occurred,
[`getPitch()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html#getPitch--)
to learn the current pitch (speed) of the player at that moment, and
[`getEffectiveTempo()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html#getEffectiveTempo--)
to find the combined effect of pitch and track BPM&mdash;the beats per minute
currently being played. You can also call
[`getBeatWithinBar()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html#getBeatWithinBar--)
to see where this beat falls within a measure of music.

If the `VirtualCdj` is active, you can also call the `Beat` object&rsquo;s
[`isTempoMaster()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/Beat.html#isTempoMaster--)
method to find out if it was sent by the device that is currently in
control of the master tempo. (If `VirtualCdj` was not started, this
method will throw an `IllegalStateException`.)

### Getting Device Details

To find some kinds of information, like which device is the tempo master,
how many beats of a track have been played, or how many beats there are
until the next cue point in a track, and any detailed information about
the tracks themselves, you need to have beat-link create a virtual
player on the network. This causes the other players to send detailed status
updates directly to beat-link, so it can interpret and keep track of
this information for you.

```java
import org.deepsymmetry.beatlink.VirtualCdj;

// ...

    VirtualCdj.getInstance().start();
```

> The Virtual player is normally created using device number `5` and the
> name `beat-link`, and announces its presence every second and a half.
> These values can be changed with
[`setDeviceNumber()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#setDeviceNumber-byte-),
[`setDeviceName()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#setDeviceName-java.lang.String-), and
[`setAnnounceInterval()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#setAnnounceInterval-int-).

As soon as it is running, you can pass any of the device announcements returned by
[`DeviceFinder.getCurrentDevices()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/DeviceFinder.html#getCurrentDevices--)
to [`VirtualCdj.getLatestStatusFor(device)`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#getLatestStatusFor-org.deepsymmetry.beatlink.DeviceAnnouncement-)
and get back the most recent status update received from that device.
The return value will either be a
[`MixerStatus`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/MixerStatus.html)
or a [`CdjStatus`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/CdjStatus.html),
containing a great deal of useful information.

> As described [above](#responding-to-beats), you can do the same thing
> with the `Beat` objects returned by [`BeatFinder`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/BeatFinder.html).

In addition to asking about individual devices, you can find out which
device is the current tempo master by calling
[`VirtualCdj.getTempoMaster()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#getTempoMaster--),
and learn the current master tempo by calling
[`VirtualCdj.getMasterTempo()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#getMasterTempo--).

If you want to be notified when either of these values change, or whenever
the player that is the current tempo master starts a new beat, you can use
[`VirtualCdj.addMasterListener()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#addMasterListener-org.deepsymmetry.beatlink.MasterListener-).

If you are building an interface that wants to display as much detail
as possible, you can request every device status update
as soon as it is received, using
[`VirtualCdj.addUpdateListener()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/VirtualCdj.html#addUpdateListener-org.deepsymmetry.beatlink.DeviceUpdateListener-).

### Getting Track Metadata

If you want to be able to retrieve details about loaded tracks, such as
the title, artist, genre, length (in seconds), key, and even artwork
images, start the `MetadataFinder`, which will also start the
`VirtualCdj` if it is not already running.

```java
import org.deepsymmetry.beatlink.data.MetadataFinder;

// ...

    MetadataFinder.getInstance().start();
```

> The safest way to successfully retrieve metadata is to configure the
> `VirtualCdj` to use a device number in the range 1 to 4, like an
> actual CDJ, using `VirtualCdj.getInstance().setDeviceNumber()` as described above.
> You can only do that if you are using fewer than 4 CDJs,
> because you need to use a number that is not being used by any actual
> CDJ. If you are using 4 actual CDJs, you will need to leave the
> `VirtualCdj` using its default number of 5, but that means the
> `MetadataFinder` will need to "borrow" one of the actual CDJ device
> numbers when it is requesting metadata. If three of the CDJs have
> loaded tracks from a media slot on the fourth, then there will be no
> device numbers available for use, and the metadata request will not
> even be attempted. Even if they have not, there is no way for
> beat-link to know if the DJ is using Link Info in a way that causes
> the metadata request to fail, or worse, causes one of the CDJs to get
> confused and stop working quite right. So if you want to work with
> metadata, to be safe, reserve a physical player number from 1 to 4
> for the exclusive use of beat-link, or have all the CDJs load tracks
> from rekordbox, rather than from each other.
>
> Alternately, you can tell the `MetadataFinder` to create a cache file
> of the metadata from a media slot when you have a convenient moment,
> and then have it use that cache file during a busy show when the
> players would have a hard time responding to queries.

Once the `MetadataFinder` is running, you can access all the metadata
for currently-loaded tracks by calling
[`MetadataFinder.getInstance().getLoadedTracks()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/MetadataFinder.html#getLoadedTracks--),
which returns a `Map` from deck references (player numbers and
hot cue numbers) to
[`TrackMetadata`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/TrackMetadata.html)
objects describing the track currently loaded in that player slot. You can
also call [`MetadataFinder.getLatestMetadataFor(int player)`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/MetadataFinder.html#getLatestMetadataFor-int-)
to find the metadata for the track loaded in the playback deck of the specified player. See
the [`TrackMetadata`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/TrackMetadata.html)
API documentation for all the details it provides.

With the `MetadataFinder` running, you can also start the `ArtFinder`,
and use its
can call its [`getLoadedArt()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/ArtFinder.html#getLoadedArt--)
or [`getLatestArtFor()`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/data/ArtFinder.html#getLatestArtFor-int-)
methods to get the artwork images associated with the tracks, if there
are any.

### Getting Other Track Information

As of version 0.3.0, much more of the database protocol has been
understood and implemented, so you can use objects in the
`org.deepsymmetry.beatlink.data` namespace to get beat grids and
track waveform data, and even to create Swing UI components to display
the preview and detailed waveform, reflecting the current playback
position.

To illustrate the kind of interface that you can now put
together from the elements offered by Beat Link, here is the Player
Status window from Beat Link Trigger:

<image src="assets/PlayerStatus.png" alt="Player Status window" width="538">

## An Example

Here is the source for `Example.java`, a small class that demonstrates
how to watch for changes related to the tempo master (and also shows
that printing
[`DeviceUpdate`](http://deepsymmetry.org/beatlink/apidocs/org/deepsymmetry/beatlink/DeviceUpdate.html)
objects can be a useful diagnostic aid):

```java
import java.util.Date;
import org.deepsymmetry.beatlink.*;

public class Example {

    public static void main(String[] args) {
        try {
            VirtualCdj.getInstance().start();
        } catch (java.net.SocketException e) {
            System.err.println("Unable to start VirtualCdj: " + e);
        }

        VirtualCdj.getInstance().addMasterListener(new MasterListener() {
                        @Override
                        public void masterChanged(DeviceUpdate update) {
                            System.out.println("Master changed at " + new Date() + ": " + update);
                        }

                        @Override
                        public void tempoChanged(double tempo) {
                            System.out.println("Tempo changed at " + new Date() + ": " + tempo);
                        }

                        @Override
                        public void newBeat(Beat beat) {
                            System.out.println("Master player beat at " + new Date() + ": " + beat);
                        }
            });

        try {
            Thread.sleep(60000);
        } catch (InterruptedException e) {
            System.out.println("Interrupted, exiting.");
        }
    }
}
```

Compiling and running this class (with the beat-link and slf4j jars on
the class path) watches and reports on tempo master activity for one
minute, producing output like this:

```
> java -cp .:beat-link.jar:slf4j-api-1.7.21.jar:slf4j-simple-1.7.21.jar Example
Master changed at Sun May 08 20:49:23 CDT 2016: CDJ status: Device 2,
 name: DJ-2000nexus, busy? true, pitch: +0.00%, track: 5, track BPM:
 128.0, effective BPM: 128.0, beat: 55, beat within bar: 3,
 cue: --.-, Playing? true, Master? true, Synced? true, On-Air? true
Tempo changed at Sun May 08 20:49:23 CDT 2016: 128.0
Tempo changed at Sun May 08 20:49:47 CDT 2016: 127.9359130859375
    [... lines omitted ...]
Tempo changed at Sun May 08 20:49:51 CDT 2016: 124.991943359375
Tempo changed at Sun May 08 20:49:51 CDT 2016: 124.927978515625
Master changed at Sun May 08 20:49:55 CDT 2016: CDJ status: Device 3,
 name: DJ-2000nexus, busy? true, pitch: -0.85%, track: 4, track BPM:
 126.0, effective BPM: 124.9, beat: 25, beat within bar: 1,
 cue: 31.4, Playing? true, Master? true, Synced? true, On-Air? true
```

## Research

This project is being developed with the help of
[dysentery](https://github.com/brunchboy/dysentery). Check that out
for details of the packets and protocol, and for ways you can help
figure out more.

## License

<img align="right" alt="Deep Symmetry" src="assets/DS-logo-bw-200-padded-left.png">

Copyright © 2016&ndash;2017 [Deep Symmetry, LLC](http://deepsymmetry.org)

Distributed under the
[Eclipse Public License 1.0](http://opensource.org/licenses/eclipse-1.0.php).
By using this software in any fashion, you are agreeing to be bound by
the terms of this license. You must not remove this notice, or any
other, from this software.
