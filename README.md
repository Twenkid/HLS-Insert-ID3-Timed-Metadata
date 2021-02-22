# HLS-Insert-ID3-Timed-Metadata

Notes by Todor Arnaudov on researching a solution for the task:

## "Insert ID3 timed metadata in an HLS stream" on Linux or Windows

* The platform is an important remark, because there are standard tools for Mac, provided by Apple, as its their technology. However doing it on Linux or Windows requires "hacks" or custom-made parsers to modify the stream.

See also the other HLS/m3u8 related repositories in Twenkid's profile:

https://github.com/Twenkid/hls-creator

https://github.com/Twenkid/HLS-M3U8-Playlist-Operations

(...)

I see the process is supposed to be straightforward with the proper Apple tools: https://jmacmullin.wordpress.com/2010/11/03/adding-meta-data-to-video-in-ios/

Without them it turned into a debugging adventure - the injector didn't work as expected... So after exploring the PHP script I discovered that it might not work because the PTS/DTS indicator flags are not 11(binary), but so far I don't know how to set them properly (if that's the only reason).

Someone mentions that reg. Apple requirement and it is in the PHP script, however the script doesn't inject to an Apple example .ts as well.

I tried many things both with the Linux shell script wrapper of the PHP/Perl injector ..., and with the original PHP-only version that is linked there https://github.com/lebougui/hls-id3tags --> https://github.com/dusterio/hlsinjector (the PHP injector is the same code, except the top lines about the author, I compared them with Merge).

Unfortunately both projects lack a sample .ts file before-after injection to test with it.

...

So I first worked both with the Linux version (on Ubuntu Studio 19.xx [a VM on VirtualBox] ) and Windows, but they both failed to insert, so I focused on the PHP only on Windows.

I found one evidence for the PTS/DTS issue:

https://video.stackexchange.com/questions/28855/pts-dts-indicator-in-pes-frame

>PTS DTS indicator in PES frame
>Asked 11 months ago
>
>I'm trying to add timed metadata into TS files and to the best of my knowledge it is only possible if the PES frame has the PTS DTS indicator with the value 2, which means that only PTS is present. From my experience this is the only case when after injecting the timed metadata, the video also works on iOS devices. Problem is that I don't know how to properly encode MP4 source files to TS files in a way that would set this kind of indicators at specific points in time. With the most basic ffmpeg command this indicator rarely appears, if at all (value 2). Through blind luck I found out that forcing key frames makes this indicator appear more often. Does anyone know how I can force PES frames with indicator value 2 at specific times during the encoding process? Or can someone clarify how even the indicator value is chosen for each PES frame? Currently I am using the following command for encoding:
>
>ffmpeg -i "input.mp4" -hls_list_size 0 -force_key_frames "expr:gte(t,n_forced*1)" -f hls -hls_time 10 -c:v h264 -profile:v main  -s 1920x1080 -b:v 2000k -c:a aac -ab 96k -ar 32000 "output_2000_.m3u8"
>See https://en.wikipedia.org/wiki/Packetized_elementary_stream for information about PES, specifically check the Optional PES header section."


Nobody has answered though.

https://en.wikipedia.org/wiki/Packetized_elementary_stream

PTS DTS indicator 2
11 = both present, 01 is forbidden, 10 = only PTS, 00 = no PTS or DTS

As suggested by the PHP code:

```
if ($debug) echo "2 bits: PTS & DTS indicator = " . (($oneByte >> 6) & 3) . "\n";
           if ((($oneByte >> 6) & 3) == 2) {
                if ($debug) echo "** Only PTS is expected\n"; #11 bin
                $ptsOnly = true;
            } else {
                // We can only parse PTS at the moment anyway
                $ptsOnly = false;
                return $response;
            }
```

(I am not sure yet what gonna happen if the bits are set to 11 by the script itself.).

The script correctly parses the metadata which I feed in:

```
E:\>E:\Wampee-3.1.0-beta-3.5\bin\php\php7.2.4\php.exe e:\injector.php  -i e:\fileSequence0.ts -m inject -o e:\fsc_meta.ts -e tag.txt --metastart 0 -d > 1.txt
```

( --metastart 0  is optional, if not given it's 0 by default)

The input is an Apple sample:

https://devstreaming-cdn.apple.com/videos/streaming/examples/img_bipbop_adv_example_ts/master.m3u8

https://devstreaming-cdn.apple.com/videos/streaming/examples/img_bipbop_adv_example_ts/v6/prog_index.m3u8

https://devstreaming-cdn.apple.com/videos/streaming/examples/img_bipbop_adv_example_ts/v6/fileSequence0.ts

Parsed 28675 MPEG TS frames with 0 errors
Total of 1 programs and 1 streams
Injected 0 frames
Finished in 618ms

tag.txt contains:
```
11 id3 1.tag
12 plaintext "BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB"
13 plaintext "CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC"
```

It is tested both with 1,2,3 and 11,12,13 for an Apple .ts, because it seems to start nominally from 10 sec, according to the viewTS graph; some discussions also suggest that:


1.tag is created by the script, taken from the debug output of the script, typed in a HEX editor:

<img src="https://github.com/Twenkid/HLS-Insert-ID3-Timed-Metadata/blob/main/i1.png">

(I tried also with premade id3 tags, e.g. 230-unicode.tag and others with lyrics).

The script reports:

"Imported 3 metadata tags" so I consider it OK.

It modifies the PMT and reports that it adds another stream_ID = 258:

```
@@@modifyFramePMT and seems to add another data stream?

10 bits: Old section length = 18
10 bits: Old program info length = 0 bytes
New program info length = 17
13 bits: Elementary PID = 257 (dec) / 0x101 (hex)
** Stream ID 258 will be used for metadata
10 bits: New section length = 55
** CRC checksum = 0xdb387cca
metastreamID = 258
...
```
However it ends up with an empty hash table that's supposed to contain timestamps for the injection section:

@@@empty(parsedResult['timestamp']!!! No timestamps, PTS/DTS

It doesn't enter the scope where the metaFrame is generated and injected in the stream file:

```
 if (!empty($parsedResult['timestamp'])) {
                  if (!$initialShift) $initialShift = ($parsedResult['timestamp'] / $clockFrequency);
                    if ($debug) echo "** Timestamp received: " . $parsedResult['timestamp'] . "\n";
                    if (count($metaData) > 0) {
                        if ($metastreamID && (($parsedResult['timestamp'] / $clockFrequency) - $initialShift) >= $metaData[0]['moment']) {
                            $metaFrame = generateMetaFrame($metaData[0]['tag'], $metastreamID, $parsedResult[
```
...

Also, viewTS graph of the output fsc_meta.ts doesn't show the presence of pid 258 stream (I have not studied the code so deep to trace that, too).

As far I understand for now, the script is supposed to add that new program_id (data stream) and the new timestamps ... Or it has to be prepared beforehand and forced (with PTS/DTS set, the question of the colleague).

I haven't found how yet, although I noticed ffmpeg parameters regarding mpegts tables or stream ids, such as:


>" -streamid output-stream-index:new-value (output)
>
>>    Assign a new stream-id value to an output stream. This option should be specified prior to the output filename to which it applies. For the situation where multiple output files exist, a streamid may be reassigned to a different value.
>
>   For example, to set the stream 0 PID to 33 and the stream 1 PID to 36 for an output mpegts file:
>
```   ffmpeg -i inurl -streamid 0:33 -streamid 1:36 out.ts"
```

However the script seems to add what it needs automatically (258 here).

...

ID3 data stream appears in ffplay

1) FFPLAY doesn't report an id3 data stream in the original .ts (as it should):

```
E:\>ffplay fileSequence0.ts
ffplay version git-2020-03-15-c467328 Copyright (c) 2003-2020 the FFmpeg develop
ers (...)
Input #0, mpegts, from 'fileSequence0.ts':    0KB sq=    0B f=0/0
  Duration: 00:00:06.00, start: 10.016667, bitrate: 3094 kb/s
  Program 1
    Stream #0:0[0x101]: Video: h264 (High) ([27][0][0][0] / 0x001B), yuv420p(pro
gressive), 1280x720 [SAR 1:1 DAR 16:9], Closed Captions, 60 fps, 60 tbr, 90k tbn
, 120 tbc
  11.81 M-V:  0.007 fd=   1 aq=    0KB vq=  370KB sq=    0B f=0/0
```

When playing the output file after the unsuccessful id3 tag insertion, ffplay displays the presence of data stream "timed_id3" (and strangely a silent sound track, which is lacking in the original)

```
E:\>ffplay fsc_meta2.ts
[mpegts @ 0000000000517400] start time for stream 1 is not set in estimate_timin
gs_from_pts
Input #0, mpegts, from 'fsc_meta2.ts':
  Duration: 00:00:06.00, start: 10.016667, bitrate: 3094 kb/s
  Program 1
    Stream #0:0[0x101]: Video: h264 (High) ([27][0][0][0] / 0x001B), yuv420p(pro
gressive), 1280x720 [SAR 1:1 DAR 16:9], Closed Captions, 60 fps, 60 tbr, 90k tbn
, 120 tbc
    Stream #0:1[0x102]: Data: timed_id3 (ID3  / 0x20334449)
Switch subtitle stream from #-1 to #-1 vq=  336KB sq=    0B f=0/0
Switch subtitle stream from #-1 to #-1 vq=  376KB sq=    0B f=0/0
Switch subtitle stream from #-1 to #-1 vq=  406KB sq=    0B f=0/0
Switch subtitle stream from #-1 to #-1 vq=  386KB sq=    0B f=0/0
  14.57 M-V:  0.000 fd=   1 aq=    0KB vq=  359KB sq=    0B f=0/0
```

However when peeking in the file with a HEX editor, ID3 is only appearing in the beginning (per file only); also the text is not there as it's not inserted and it seems as stuffed with placeholders FF.

<img src="https://github.com/Twenkid/HLS-Insert-ID3-Timed-Metadata/blob/main/i2.png">

...

If injecting the produced stream again - another id3 stream appears in the ffplay intro:

```
E:\>ffplay E:\fsc_meta4.ts
Input #0, mpegts, from 'E:\fsc_meta4.ts':
  Duration: 00:00:06.00, start: 10.016667, bitrate: 3094 kb/s
  Program 1
    Stream #0:0[0x101]: Video: h264 (High) ([27][0][0][0] / 0x001B), yuv420p(pro
gressive), 1280x720 [SAR 1:1 DAR 16:9], Closed Captions, 60 fps, 60 tbr, 90k tbn
, 120 tbc
    Stream #0:1[0x102]: Data: timed_id3 (ID3  / 0x20334449)
    Stream #0:2[0x103]: Data: timed_id3 (ID3  / 0x20334449)
  11.48 M-V:  0.004 fd=   1 aq=    0KB vq=  393KB sq=    0B f=0/0
```

The Hex view:

<img src="https://github.com/Twenkid/HLS-Insert-ID3-Timed-Metadata/blob/main/i3.png">

Still there's only one Pid = 257


...

31.10.2020
