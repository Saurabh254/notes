# HLS (HTTP Live Streaming)

HLS is how most video on the internet reaches your screen. This guide explains what it is, how it works, what bitrates and adaptive streaming mean, and how to build HLS yourself with FFmpeg.

## Contents

1. What is HLS
2. How HLS works
3. Playlists and segments
4. Segment formats: TS vs fMP4
5. Bitrates
6. Adaptive bitrate streaming
7. VOD vs Live
8. Processing with FFmpeg
9. Serving and playing HLS
10. Latency and Low Latency HLS
11. Troubleshooting
12. Cheat sheet
13. Glossary

## 1. What is HLS

**HLS** stands for **HTTP Live Streaming**. Apple created it in 2009. It is now documented in RFC 8216 and supported almost everywhere.

The idea is simple:

1. Cut a video into many small files called **segments** (usually 2 to 10 seconds each).
2. Write a plain text file called a **playlist** that lists those segments in order.
3. Put everything on a normal web server.
4. The player reads the playlist, downloads segments one by one, and plays them.

That is it. No special streaming server. Just files over HTTP.

The word "Live" in the name is misleading. HLS works for both live streams and videos on demand (VOD).

### Why people use it

* Runs on any web server or CDN. Cheap and easy to scale.
* Works over port 80 or 443, so firewalls rarely block it.
* Adaptive. Video quality changes with the viewer's network speed.
* Plays on iPhone, Android, browsers, smart TVs, and set top boxes.
* Supports encryption, subtitles, multiple audio languages, and ads.

### HLS vs a plain MP4 file

| | Plain MP4 | HLS |
|---|---|---|
| Delivery | One big file | Many small segments |
| Quality switching | No | Yes, automatic |
| Live streaming | No | Yes |
| Seeking | Needs range requests | Jump to any segment |
| CDN friendly | Okay | Excellent |
| Setup effort | Very low | Medium |

### HLS vs MPEG DASH

DASH is the open standard alternative. Same core idea, different playlist format (an XML file called MPD). HLS wins on Apple device support. DASH wins on some codec flexibility. Many teams ship both, using one set of fMP4 segments (see section 4).

## 2. How HLS works

The full journey, from raw video to playback:

```
Source video (mp4, mov, camera feed)
        |
        v
Encoder and packager (FFmpeg)
  * makes several quality versions
  * cuts each into segments
  * writes playlists
        |
        v
Web server or CDN
  master.m3u8
  720p/index.m3u8, 720p/seg_000.ts, ...
  480p/index.m3u8, 480p/seg_000.ts, ...
        |
        v
Player (Safari, hls.js, ExoPlayer, VLC)
  1. downloads master.m3u8
  2. picks a quality
  3. downloads that quality's playlist
  4. downloads segments and plays them
  5. keeps measuring speed and switches quality when needed
```

### What the player does, step by step

1. The player requests the **master playlist**. It learns which qualities exist.
2. It picks a starting quality. Often a middle or low one so playback starts fast.
3. It requests the **media playlist** for that quality.
4. It downloads the first few segments and starts playing.
5. While playing, it keeps downloading the next segments ahead of time. This is the **buffer**.
6. After each segment, it checks download speed. If the network got faster or slower, it switches quality at the next segment.

## 3. Playlists and segments

### File types

| File | What it is |
|---|---|
| `.m3u8` | Playlist. Plain UTF 8 text. |
| `.ts` | Segment in MPEG Transport Stream format. |
| `.m4s` | Segment in fragmented MP4 format. |
| `init.mp4` | Init file for fMP4. Holds codec setup info. |
| `.key` | Encryption key (only if you encrypt). |

### Two kinds of playlists

**1. Master playlist** (also called multivariant playlist). Lists all available qualities.

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=900000,RESOLUTION=640x360,CODECS="avc1.4d401e,mp4a.40.2"
360p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1600000,RESOLUTION=854x480,CODECS="avc1.4d401f,mp4a.40.2"
480p/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720,CODECS="avc1.4d4020,mp4a.40.2"
720p/index.m3u8
```

Each `EXT-X-STREAM-INF` line describes one quality. The next line is the URL of its media playlist.

**2. Media playlist.** Lists the segments of one quality.

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:6
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXTINF:6.000,
seg_000.ts
#EXTINF:6.000,
seg_001.ts
#EXTINF:4.200,
seg_002.ts
#EXT-X-ENDLIST
```

### Common tags

| Tag | Meaning |
|---|---|
| `#EXTM3U` | First line. Marks the file as an extended M3U playlist. |
| `#EXT-X-VERSION` | HLS protocol version used by the playlist. |
| `#EXT-X-TARGETDURATION` | Max segment length in seconds (rounded up). |
| `#EXT-X-MEDIA-SEQUENCE` | Number of the first segment in the list. Grows in live streams. |
| `#EXTINF` | Exact duration of the next segment. |
| `#EXT-X-PLAYLIST-TYPE` | `VOD` (never changes) or `EVENT` (only grows). |
| `#EXT-X-ENDLIST` | Stream is finished. No more segments will come. |
| `#EXT-X-STREAM-INF` | One quality option in a master playlist. |
| `#EXT-X-MEDIA` | Separate audio or subtitle tracks (languages). |
| `#EXT-X-KEY` | Encryption method and key URL. |
| `#EXT-X-MAP` | Points to the init segment (fMP4 only). |
| `#EXT-X-INDEPENDENT-SEGMENTS` | Every segment can be decoded on its own. |
| `#EXT-X-DISCONTINUITY` | Something changed here (codec, timestamps, ad break). |
| `#EXT-X-PROGRAM-DATE-TIME` | Real world clock time of a segment. |

### Segment length tradeoffs

| Shorter segments (2 to 4 s) | Longer segments (6 to 10 s) |
|---|---|
| Faster quality switching | Fewer HTTP requests |
| Lower live latency | Better compression (fewer keyframes) |
| More requests and bigger playlists | Slower to react to network changes |

Apple recommends about **6 seconds**. It is a solid default.

## 4. Segment formats: TS vs fMP4

| | MPEG TS (`.ts`) | Fragmented MP4 (`.m4s`) |
|---|---|---|
| Age | Original HLS format | Added in 2016 |
| Codecs | H.264, H.265, AAC, AC3 | H.264, H.265, AV1, AAC, more |
| Overhead | Higher (about 5 to 10 percent) | Lower |
| Works with DASH | No | Yes (CMAF) |
| Init file needed | No | Yes (`init.mp4`) |
| Best for | Maximum old device support | Modern setups, HEVC, low latency |

**Rule of thumb:** use TS for simple and maximum compatibility. Use fMP4 if you want HEVC, AV1, Low Latency HLS, or one set of files for both HLS and DASH.

## 5. Bitrates

### What is bitrate

**Bitrate** is how many bits of data one second of video or audio uses. Measured in **kbps** (kilobits per second) or **Mbps** (megabits per second).

* More bitrate means more detail and a bigger file.
* Less bitrate means more blur and compression artifacts, but a smaller file.

### Quick file size formula

```
size in MB = bitrate in Mbps x seconds / 8
```

Example: 2.8 Mbps video, 10 minutes (600 s):

```
2.8 x 600 / 8 = 210 MB
```

### What decides how much bitrate you need

* **Resolution.** More pixels need more bits.
* **Frame rate.** 60 fps needs roughly 1.5x the bitrate of 30 fps.
* **Codec.** HEVC needs about 30 to 40 percent less than H.264 for similar quality.
* **Content.** Sports and action need more. Slideshows and talking heads need less.

### A typical bitrate ladder (H.264)

A **ladder** is the set of qualities you offer. This is a good starting point. Tune it for your content.

| Name | Resolution | Video bitrate | Audio bitrate |
|---|---|---|---|
| 1080p | 1920x1080 | 5000 kbps | 128 kbps |
| 720p | 1280x720 | 2800 kbps | 128 kbps |
| 480p | 854x480 | 1400 kbps | 96 kbps |
| 360p | 640x360 | 800 kbps | 64 kbps |
| 240p | 426x240 | 400 kbps | 64 kbps |

Tips:

* Steps between rungs should be about 1.5x to 2x. Too close wastes storage. Too far feels jumpy.
* Always include a low rung so people on bad mobile networks can still watch.
* Do not upscale. If the source is 720p, do not make a 1080p rung.
* Storage cost is the **sum** of all rungs. A 5 rung ladder uses roughly 2x the storage of the top rung alone.

### Bitrate modes

| Mode | Meaning | HLS use |
|---|---|---|
| **CBR** | Constant bitrate | Predictable, but wastes bits on easy scenes |
| **VBR** | Variable bitrate | Efficient, but can spike |
| **Capped VBR** | VBR with a ceiling (`-maxrate`, `-bufsize`) | **Best for HLS.** |

Capped VBR matters because the `BANDWIDTH` in your master playlist is a promise about the **peak**. If your stream spikes far above it, players stall.

### BANDWIDTH in the master playlist

`BANDWIDTH` is the **peak** bitrate of video plus audio, in bits per second. Players use it to decide if they can afford that quality. Some playlists also add `AVERAGE-BANDWIDTH`.

## 6. Adaptive bitrate streaming

### The problem

Viewer networks change all the time. Wifi gets weak. A train enters a tunnel. A single fixed quality would either buffer a lot or look bad when it does not need to.

### The solution

**Adaptive Bitrate Streaming (ABR)** means the player switches quality on the fly.

How it works:

1. Every quality is cut at the **same time points**, so segment 12 of 720p covers the same seconds as segment 12 of 480p.
2. After each segment download, the player measures **throughput** (how fast it downloaded).
3. It also watches the **buffer level** (how many seconds are queued up).
4. It picks the highest quality whose `BANDWIDTH` fits under the measured speed, with a safety margin (often 70 to 80 percent of throughput).
5. It switches at the **next segment boundary**.

### Simple example

| Moment | Measured speed | Buffer | Player picks |
|---|---|---|---|
| Start | unknown | 0 s | 360p (safe start) |
| 10 s | 8 Mbps | 12 s | 1080p |
| 40 s | 1.2 Mbps | 5 s | 480p |
| 60 s | 1.0 Mbps | 2 s | 360p (avoid stall) |

### What the content creator must do right

* **Align keyframes.** Every segment must start with a keyframe (IDR frame), and all qualities must cut at the same moments. Otherwise switching causes glitches.
* Use the same segment duration across all qualities.
* Use `independent_segments` so each segment can be decoded alone.
* Keep audio the same or compatible across qualities for smooth switching.

### Who makes the decision

The **player** decides, not the server. The server just hosts files. Different players use different algorithms (throughput based, buffer based, or hybrids). You can tune some of this in hls.js and ExoPlayer.

## 7. VOD vs Live

### VOD (video on demand)

* The full playlist exists upfront.
* Ends with `#EXT-X-ENDLIST`.
* Never changes. Cache it hard on the CDN.
* Users can seek anywhere.

### Live

* The encoder keeps producing new segments.
* The playlist is **updated** every few seconds. No `ENDLIST` until the stream ends.
* Usually a **sliding window**: only the latest N segments are listed. Old ones are deleted. `EXT-X-MEDIA-SEQUENCE` increases as the window moves.
* The player reloads the playlist about every target duration.
* Short cache times on playlists (about 1 to 2 seconds or less). Segments can cache longer.

Live playlist example:

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:4
#EXT-X-MEDIA-SEQUENCE:1520
#EXTINF:4.000,
seg_1520.ts
#EXTINF:4.000,
seg_1521.ts
#EXTINF:4.000,
seg_1522.ts
```

### EVENT playlists

Type `EVENT` means: segments only get added, never removed. Good for a live stream you want people to rewind, and then keep as VOD after it ends.

## 8. Processing with FFmpeg

FFmpeg is the standard tool for making HLS.

### 8.1 Install and inspect

```bash
# macOS
brew install ffmpeg

# Ubuntu or Debian
sudo apt install ffmpeg

# check
ffmpeg -version
```

Look at your source file first:

```bash
ffprobe -v error \
  -show_entries stream=codec_name,codec_type,width,height,r_frame_rate,bit_rate \
  -of default=nw=1 input.mp4
```

Note the frame rate. You need it to set keyframe spacing.

### 8.2 Simplest HLS (one quality)

```bash
mkdir -p out

ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_filename "out/seg_%03d.ts" \
  out/index.m3u8
```

You get `out/index.m3u8` and `out/seg_000.ts`, `seg_001.ts`, and so on.

### 8.3 The flags, explained

| Flag | What it does |
|---|---|
| `-i input.mp4` | Input file (or URL, or stream). |
| `-c:v libx264` | Encode video as H.264. |
| `-c:a aac` | Encode audio as AAC. |
| `-b:v 2800k` | Target video bitrate. |
| `-maxrate 3000k` | Peak bitrate allowed. |
| `-bufsize 4200k` | Size of the rate control buffer. Usually 1.5x to 2x maxrate. |
| `-preset medium` | Speed vs compression. Slower presets make smaller files at the same quality. |
| `-g 48` | Keyframe every 48 frames. |
| `-keyint_min 48` | Minimum gap between keyframes. Same value locks the interval. |
| `-sc_threshold 0` | Do not insert extra keyframes on scene changes. Keeps all qualities aligned. |
| `-hls_time 6` | Target segment length in seconds. |
| `-hls_playlist_type vod` | Make a complete VOD playlist with `ENDLIST`. |
| `-hls_list_size 0` | Keep all segments in the playlist. (Default for live is 5.) |
| `-hls_segment_filename` | Naming pattern for segment files. |
| `-hls_segment_type` | `mpegts` (default) or `fmp4`. |
| `-hls_flags` | Extra behaviors, joined with `+`. Examples: `independent_segments`, `delete_segments`, `append_list`, `omit_endlist`. |
| `-master_pl_name` | File name of the master playlist when making many qualities. |
| `-var_stream_map` | Groups video and audio streams into qualities. |
| `-hls_key_info_file` | Turns on AES 128 encryption. |

### 8.4 Keyframes and segment length

`-hls_time 6` is a **target**. FFmpeg can only cut at a keyframe. So if your keyframes come every 10 seconds, your segments will be 10 seconds no matter what.

Fix: set keyframe spacing to divide evenly into the segment length.

```
-g = frame rate x keyframe seconds
```

Examples:

| Frame rate | Keyframe every 2 s | Flags |
|---|---|---|
| 24 fps | 48 frames | `-g 48 -keyint_min 48` |
| 25 fps | 50 frames | `-g 50 -keyint_min 50` |
| 30 fps | 60 frames | `-g 60 -keyint_min 60` |

With 2 second keyframes and 6 second segments, every segment holds exactly 3 GOPs. Clean and predictable.

Alternative that ignores the frame rate:

```bash
-force_key_frames "expr:gte(t,n_forced*2)"
```

### 8.5 No re encoding (fast, no quality loss)

If your video is already H.264 and AAC, just repackage it:

```bash
ffmpeg -i input.mp4 \
  -c copy \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_filename "out/seg_%03d.ts" \
  out/index.m3u8
```

Takes seconds. The catch: cuts happen only at existing keyframes, so segment lengths may be uneven. And you get only one quality.

### 8.6 Multi bitrate HLS with a master playlist

This is the real deal. One command makes three qualities plus the master playlist.

```bash
mkdir -p out/v0 out/v1 out/v2

ffmpeg -i input.mp4 \
  -filter_complex "[0:v]split=3[v1][v2][v3]; \
    [v1]scale=-2:1080[v1out]; \
    [v2]scale=-2:720[v2out]; \
    [v3]scale=-2:480[v3out]" \
  -map "[v1out]" -c:v:0 libx264 -b:v:0 5000k -maxrate:v:0 5350k -bufsize:v:0 7500k \
  -map "[v2out]" -c:v:1 libx264 -b:v:1 2800k -maxrate:v:1 2996k -bufsize:v:1 4200k \
  -map "[v3out]" -c:v:2 libx264 -b:v:2 1400k -maxrate:v:2 1498k -bufsize:v:2 2100k \
  -map a:0 -c:a:0 aac -b:a:0 128k -ac 2 \
  -map a:0 -c:a:1 aac -b:a:1 128k -ac 2 \
  -map a:0 -c:a:2 aac -b:a:2 96k  -ac 2 \
  -preset veryfast -pix_fmt yuv420p -profile:v main \
  -g 48 -keyint_min 48 -sc_threshold 0 \
  -f hls \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_flags independent_segments \
  -master_pl_name master.m3u8 \
  -hls_segment_filename "out/v%v/seg_%03d.ts" \
  -var_stream_map "v:0,a:0 v:1,a:1 v:2,a:2" \
  "out/v%v/index.m3u8"
```

How to read it:

* `split=3` copies the decoded video into three branches.
* Each branch gets scaled. `-2` keeps the aspect ratio and an even width.
* Each `-map` picks one scaled video. The `:0`, `:1`, `:2` suffixes are output stream numbers.
* The audio is mapped three times, once per quality, so each quality has its own audio.
* `%v` in file names becomes the variant number (0, 1, 2).
* `-var_stream_map` says: quality 0 is video 0 plus audio 0, and so on.
* `-profile:v main` and `-pix_fmt yuv420p` keep it compatible with old devices.

To add a rung, add another split output, another video `-map`, another audio `-map`, and another group in `-var_stream_map`.

Result:

```
out/
  master.m3u8
  v0/
    index.m3u8
    seg_000.ts
    seg_001.ts
    ...
  v1/
    index.m3u8
    seg_000.ts
    ...
  v2/
    index.m3u8
    seg_000.ts
    ...
```

The generated `master.m3u8` looks roughly like this:

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-STREAM-INF:BANDWIDTH=5500000,RESOLUTION=1920x1080,CODECS="avc1.4d4028,mp4a.40.2"
v0/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720,CODECS="avc1.4d401f,mp4a.40.2"
v1/index.m3u8
#EXT-X-STREAM-INF:BANDWIDTH=1600000,RESOLUTION=854x480,CODECS="avc1.4d401f,mp4a.40.2"
v2/index.m3u8
```

Want friendly folder names like `1080p`, `720p`? Add `name:` to each group:

```bash
-var_stream_map "v:0,a:0,name:1080p v:1,a:1,name:720p v:2,a:2,name:480p"
```

and use `%v` in the paths, which then expands to the name.

### 8.7 fMP4 segments instead of TS

```bash
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -g 48 -keyint_min 48 -sc_threshold 0 \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_segment_type fmp4 \
  -hls_fmp4_init_filename init.mp4 \
  -hls_segment_filename "out/seg_%03d.m4s" \
  out/index.m3u8
```

The playlist now has an `#EXT-X-MAP:URI="init.mp4"` line. Add `-tag:v hvc1` if you encode HEVC with `libx265`, so Apple devices accept it.

### 8.8 Audio only HLS (podcasts, radio)

```bash
ffmpeg -i input.mp3 \
  -vn -c:a aac -b:a 128k \
  -hls_time 10 \
  -hls_playlist_type vod \
  -hls_segment_filename "out/audio_%03d.ts" \
  out/audio.m3u8
```

### 8.9 Live streaming

Take any live input and produce a rolling playlist:

```bash
ffmpeg -i rtmp://your-server/live/stream \
  -c:v libx264 -preset veryfast -b:v 3000k -maxrate 3200k -bufsize 6000k \
  -g 60 -keyint_min 60 -sc_threshold 0 \
  -c:a aac -b:a 128k \
  -f hls \
  -hls_time 4 \
  -hls_list_size 6 \
  -hls_flags delete_segments+independent_segments \
  -hls_segment_filename "live/seg_%05d.ts" \
  live/index.m3u8
```

What matters:

* `-hls_list_size 6` keeps the 6 newest segments in the playlist.
* `delete_segments` removes old files from disk so it does not fill up.
* There is no `ENDLIST` while the stream runs.
* `-preset veryfast` (or faster) so encoding keeps up in real time.
* To record the whole stream as VOD later, use `-hls_playlist_type event` and `-hls_list_size 0`.

Other live inputs: a webcam, a screen capture, an SRT stream, or another HLS URL.

### 8.10 Encryption (AES 128)

```bash
# 1. make a random key
openssl rand 16 > enc.key

# 2. make the key info file (3 lines)
#    line 1: URL the player will use to fetch the key
#    line 2: local path to the key file
#    line 3: IV in hex (optional)
echo "https://example.com/keys/enc.key" > enc.keyinfo
echo "enc.key" >> enc.keyinfo
openssl rand -hex 16 >> enc.keyinfo

# 3. encode
ffmpeg -i input.mp4 \
  -c:v libx264 -c:a aac \
  -hls_time 6 \
  -hls_playlist_type vod \
  -hls_key_info_file enc.keyinfo \
  -hls_segment_filename "out/seg_%03d.ts" \
  out/index.m3u8
```

Honest note: AES 128 stops casual downloading. It is **not DRM**. Anyone who can reach the key URL can decrypt. For real protection, use a DRM system (FairPlay, Widevine, PlayReady) and keep the key URL behind authentication.

### 8.11 Subtitles

HLS wants subtitles as WebVTT files, referenced from the master playlist:

```m3u8
#EXT-X-MEDIA:TYPE=SUBTITLES,GROUP-ID="subs",NAME="English",LANGUAGE="en",URI="subs/en/index.m3u8"
#EXT-X-STREAM-INF:BANDWIDTH=3000000,RESOLUTION=1280x720,SUBTITLES="subs"
720p/index.m3u8
```

You can also write the extra `EXT-X-MEDIA` lines by hand into the master playlist FFmpeg generated. Same trick works for extra audio languages (`TYPE=AUDIO`).

### 8.12 Hardware encoding

Software `libx264` is the best quality per bit. For speed (especially live), swap the encoder:

| Hardware | Encoder |
|---|---|
| NVIDIA GPU | `h264_nvenc` |
| Intel Quick Sync | `h264_qsv` |
| Apple Silicon or Mac | `h264_videotoolbox` |
| AMD or Linux VAAPI | `h264_vaapi` |

Hardware encoders are faster but usually need more bitrate for the same quality.

### 8.13 Convert HLS back to MP4

```bash
ffmpeg -i "https://example.com/video/master.m3u8" \
  -c copy -bsf:a aac_adtstoasc \
  output.mp4
```

For a master playlist FFmpeg picks the best quality by default. Use `-map` to choose a specific one.

### 8.14 Check your result

```bash
# play it quickly
ffplay out/v1/index.m3u8

# see the segment durations
grep EXTINF out/v1/index.m3u8
```

## 9. Serving and playing HLS

### Serve it

Any static file server works. For a quick local test:

```bash
cd out
python3 -m http.server 8000
```

Then open `http://localhost:8000/master.m3u8` in a player.

### Correct MIME types

| Extension | MIME type |
|---|---|
| `.m3u8` | `application/vnd.apple.mpegurl` (or `application/x-mpegURL`) |
| `.ts` | `video/mp2t` |
| `.m4s` | `video/iso.segment` (or `video/mp4`) |
| `.mp4` (init) | `video/mp4` |

### CORS

If your page and your video live on different domains, the video server must send:

```
Access-Control-Allow-Origin: *
```

(or your exact site origin). Without it, browser players fail with a CORS error.

### Caching

| File | Cache time |
|---|---|
| VOD segments and playlists | Long (days to a year). They never change. |
| Live segments | Medium (a few minutes). Unique names, never change. |
| Live playlist | Very short (about half the target duration or less). |
| Master playlist | Medium for VOD, short for live. |

### Play it

**Safari, iOS, macOS** play HLS natively:

```html
<video src="https://example.com/master.m3u8" controls></video>
```

**Chrome, Firefox, Edge** need a library such as **hls.js**:

```html
<video id="video" controls></video>
<script src="https://cdn.jsdelivr.net/npm/hls.js@1"></script>
<script>
  const video = document.getElementById("video");
  const url = "https://example.com/master.m3u8";

  if (video.canPlayType("application/vnd.apple.mpegurl")) {
    video.src = url;               // native HLS (Safari)
  } else if (Hls.isSupported()) {
    const hls = new Hls();
    hls.loadSource(url);
    hls.attachMedia(video);
  }
</script>
```

**Other players:** VLC, `ffplay`, ExoPlayer (Android), AVPlayer (iOS), Video.js, Shaka Player, and most smart TV platforms.

## 10. Latency and Low Latency HLS

### Why plain HLS is slow

A player typically starts about **3 segments** behind the live edge. With 6 second segments that is 18 seconds. Add encoding, upload, and CDN time, and total delay is often **20 to 30 seconds**.

### Ways to reduce it

| Method | Latency |
|---|---|
| Normal HLS, 6 s segments | 20 to 30 s |
| Shorter segments (2 s) | 8 to 12 s |
| **LL HLS** (Low Latency HLS) | 2 to 5 s |

### What LL HLS adds

* **Partial segments.** Tiny chunks (about 200 to 500 ms) published before the full segment is done.
* **Blocking playlist reload.** The player asks for the next playlist version and the server holds the request until it is ready. No wasteful polling.
* **Preload hints.** The playlist tells the player which file is coming next.
* Needs fMP4 segments and a server and CDN that support it.

FFmpeg alone is not ideal for LL HLS. Use a tool built for it (for example Shaka Packager, Bento4, nginx modules, or a managed streaming service).

If you need under 1 second, HLS is the wrong tool. Look at WebRTC.

## 11. Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Browser says CORS error | Missing CORS header | Add `Access-Control-Allow-Origin` on the server |
| Playlist downloads as a file | Wrong MIME type | Set MIME type for `.m3u8` |
| 404 on segments | Wrong relative paths | Check the segment URLs inside the playlist |
| Uneven or huge segments | Keyframes too far apart | Set `-g`, `-keyint_min`, `-sc_threshold 0` |
| Quality never changes | Only one rung, or BANDWIDTH values wrong | Check master playlist, check real peak bitrates |
| Glitches when switching quality | Keyframes not aligned across rungs | Same `-g` and fps on all rungs, `independent_segments` |
| Stuck on start or black screen | First segment lacks a keyframe | Re encode with fixed GOP |
| Audio out of sync | Mixed timestamps or odd source | Add `-async 1` or re encode audio, check source |
| Plays in VLC but not Safari | Unsupported profile or pixel format | Use `-pix_fmt yuv420p -profile:v main`, tag HEVC as `hvc1` |
| Live stream keeps stalling | Encoder too slow, or playlist cached | Use a faster `-preset`, set short cache on live playlist |
| Live playlist looks stale | CDN caching the playlist | Lower playlist cache TTL |
| Disk fills up on live | Old segments not removed | Add `delete_segments` |
| Mobile viewers buffer a lot | No low rung | Add a 240p or 360p rung |

## 12. Cheat sheet

**Remember these:**

* HLS equals playlist (`.m3u8`) plus segments (`.ts` or `.m4s`) over HTTP.
* Master playlist lists qualities. Media playlist lists segments.
* The **player** picks quality. The server is dumb.
* Keyframes must line up at segment starts, on every rung.
* `BANDWIDTH` is peak bits per second, video plus audio.
* Segment length of 6 seconds is a good default.
* Use capped VBR (`-b:v`, `-maxrate`, `-bufsize`).
* Set `-g = fps x keyframe seconds`, and `-sc_threshold 0`.
* Use CORS and correct MIME types.
* VOD playlists cache forever. Live playlists barely cache.

**Starter command (copy and edit):**

```bash
ffmpeg -i input.mp4 -c:v libx264 -c:a aac \
  -g 48 -keyint_min 48 -sc_threshold 0 \
  -hls_time 6 -hls_playlist_type vod \
  -hls_segment_filename "out/seg_%03d.ts" out/index.m3u8
```

## 13. Glossary

| Term | Meaning |
|---|---|
| **ABR** | Adaptive bitrate. Player switches quality on its own. |
| **Bitrate** | Data used per second of media. |
| **Buffer** | Seconds of video already downloaded ahead of playback. |
| **CDN** | Network of servers that caches and delivers files close to viewers. |
| **CMAF** | Common media format. fMP4 segments shared by HLS and DASH. |
| **Codec** | Method to compress video or audio (H.264, HEVC, AAC). |
| **GOP** | Group of pictures. Frames between two keyframes. |
| **IDR frame** | A keyframe a decoder can start from. |
| **Keyframe** | A full picture frame. Other frames store only changes. |
| **Ladder** | The set of quality levels you offer. |
| **LL HLS** | Low Latency HLS. 2 to 5 seconds delay. |
| **Master playlist** | Playlist that lists all qualities. |
| **Media playlist** | Playlist that lists segments of one quality. |
| **MPD** | DASH's manifest file. |
| **Rendition** | One quality version of the stream (also called a variant). |
| **Segment** | One small chunk of the video file. |
| **TS** | MPEG Transport Stream. Original HLS segment format. |
| **VOD** | Video on demand. A finished video, not a live stream. |
| **WebVTT** | Text subtitle format used by HLS. |
