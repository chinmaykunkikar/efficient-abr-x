# Experiments for building efficient ABR ladders

Working on [Vide](https://github.com/chinmaykunkikar/vide-amplify) has been quite an experience working on ReactJS and AWS Amplify. But more interestingly, how the transcoder really works. The "[transcoder](https://github.com/chinmaykunkikar/vide-amplify/blob/f433e6b954781839674eaec277afea8ae54b8e00/amplify/backend/function/videtranscoder/src/hls-encoder.sh)" is just a bash script that is taken from [maitrungduc1410's gist](https://gist.github.com/maitrungduc1410/9c640c61a7871390843af00ae1d8758e#file-create-vod-hls-sh) to make the project run. The script intelligently generates all the ffmpeg options to run that will spit out all the MPEG-2 `.ts` files  and corresponding `.m3u8` playlists . This produces a Constant Bitrate (CBR) ladder.
Following is such a generated command by the script when given `video.mp4` as input file.

```bash
ffmpeg -hide_banner -y -i video.mp4 \
-c:a aac -ar 48000 -c:v h264 -profile:v main -crf 19 -sc_threshold 0 -g 50 -keyint_min 50 -hls_time 10 -hls_playlist_type vod -vf scale=w=426:h=-2 -b:v 400k -maxrate 428k -bufsize 600k -b:a 0k -hls_segment_filename video/240p_%03d.ts video/240p.m3u8 \
-c:a aac -ar 48000 -c:v h264 -profile:v main -crf 19 -sc_threshold 0 -g 50 -keyint_min 50 -hls_time 10 -hls_playlist_type vod -vf scale=w=640:h=-2 -b:v 800k -maxrate 856k -bufsize 1200k -b:a 0k -hls_segment_filename video/360p_%03d.ts video/360p.m3u8 \
-c:a aac -ar 48000 -c:v h264 -profile:v main -crf 19 -sc_threshold 0 -g 50 -keyint_min 50 -hls_time 10 -hls_playlist_type vod -vf scale=w=842:h=-2 -b:v 1400k -maxrate 1498k -bufsize 2100k -b:a 0k -hls_segment_filename video/480p_%03d.ts video/480p.m3u8 \
-c:a aac -ar 48000 -c:v h264 -profile:v main -crf 19 -sc_threshold 0 -g 50 -keyint_min 50 -hls_time 10 -hls_playlist_type vod -vf scale=w=1280:h=-2 -b:v 2800k -maxrate 2996k -bufsize 4200k -b:a 0k -hls_segment_filename video/720p_%03d.ts video/720p.m3u8 \
-c:a aac -ar 48000 -c:v h264 -profile:v main -crf 19 -sc_threshold 0 -g 50 -keyint_min 50 -hls_time 10 -hls_playlist_type vod -vf scale=w=1920:h=-2 -b:v 5000k -maxrate 5350k -bufsize 7500k -b:a 0k -hls_segment_filename video/1080p_%03d.ts video/1080p.m3u8
```
Right away it is apparent that it is not very optimal. It generates almost the same command to run for the next bitrate. The same can be achieved by the following command with the help of new (not really) ffmpeg features like map. Less repetitions.

```bash
ffmpeg -hide_banner -re -i video.mp4 -map 0:v:0 -map 0:v:0 -map 0:v:0 -map 0:v:0 \
-c:v h264 -profile:v main -crf 20 -sc_threshold 0 -g 50 -keyint_min 50 \
-filter:v:0 scale=w=640:h=-2 -maxrate:v:0 856k -bufsize:v:0 1200k \
-filter:v:1 scale=w=842:h=-2 -maxrate:v:1 1498k -bufsize:v:1 2100k \
-filter:v:2 scale=w=1280:h=-2 -maxrate:v:2 2996k -bufsize:v:2 4200k \
-filter:v:3 scale=w=1920:h=-2 -maxrate:v:3 5350k -bufsize:v:3 7500k \
-var_stream_map "v:0 v:1 v:2 v:3" -hls_time 10 \
-hls_playlist_type vod -master_pl_name master.m3u8 \
-hls_segment_filename video1/seg_%v_%03d.ts video1/pl_%v.m3u8
```

**So optimize the script to get the desired ffmpeg options and get it over with. Right?** \
I would, but extensive reading, and hours of YouTube videos watching has put new ideas on my mind.

#### Why this repo?
In this repo, I want to play around with stuff like building [Netflix's Per Title Encoding](https://netflixtechblog.com/per-title-encode-optimization-7e99442b62a2) and advance to Context Aware Encoding that is their current technology.

I'll mostly document my ideas about -
* ABR
* Playing around with FFMPEG options for generating more efficient ABR ladders.
* Further understanding of the HLS spec.

Also,
* Build a frontend app to visualize stats and graphs of the currently playing video.
* Also how various optimizations compare with each other in terms of quality and performance.

#### Useful resources -
* https://developer.apple.com/documentation/http_live_streaming/hls_authoring_specification_for_apple_devices - HLS Authoring Specification for Apple Devices
* https://www.dacast.com/blog/adaptive-bitrate-streaming
* https://www.mazsystems.com/en/blog/top-3-best-practices-for-adaptive-bitrate-streaming
* https://streaminglearningcenter.com/learning/beginners-guide-to-adaptive-bitrate-technologie.html
* https://streaminglearningcenter.com/learning/the-evolving-encoding-ladder-what-you-need-to-know.html (I love this site)
* https://developer.att.com/video-optimizer/docs/best-practices/adaptive-bitrate-video-streaming
* https://bento-video.github.io/ - Amazing casestudy
* https://mux.com/per-title-encoding/ - Per Title encoding overview
* https://websites.fraunhofer.de/video-dev/per-title-encoding/ - Per title encoding in action
* https://www.streamingmedia.com/Authors/4247-Jan-Ozer.htm - Jan Ozer's articles
* [Paper] https://arxiv.org/pdf/2102.04550.pdf - Efficient Bitrate Ladder Construction for Content-Optimized Adaptive Video Streaming 
* [Paper] https://arxiv.org/pdf/2103.07564.pdf - VMAF-based Bitrate Ladder Estimation for Adaptive Streaming
* [Paper] https://dl.acm.org/doi/10.1145/3210424.3210436 - Optimal Design of Encoding Profiles for ABR Streaming
