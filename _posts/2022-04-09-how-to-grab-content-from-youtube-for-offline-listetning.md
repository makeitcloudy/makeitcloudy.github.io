---
layout: post
title: "How to grab content from youtube for offline listening"
permalink: "/how-to-grab-content-from-youtube-for-offline-listening/"
subtitle: "You don't need to have premium plan"
cover-img: /assets/img/how-to-make-use-of-wired-ethernet-for-your-mobile/img-cover.jpg
thumbnail-img: /assets/img/how-to-make-use-of-wired-ethernet-for-your-mobile/img-thumb.jpg
share-img: /assets/img/how-to-make-use-of-wired-ethernet-for-your-mobile/img-cover.jpg
tags: [Audio ,Misc]
categories: [Audio ,Misc]
---
In case you are interested on listening your favourite podcasts without generating LTE/wireless traffic, when commuting or resting.

## Prerequisities

+ Download and install [ffmpeg](https://ffmpeg.org/)
+ Configure environmental variables so the ffmpeg and yt-dlp are directly available for your convinience
```powershell
$env:PATH
```

## Howto

This can be used for youtube, but also for other network streams, provided you know the path of the media stream, which can quite easily be identified with developer tools comming from the web browser.

* [youtube-dl](https://github.com/ytdl-org/youtube-dl)
* [yt-dlp](https://github.com/yt-dlp/yt-dlp)

## Where to find some usefull details

+ Github and [stackoverflow](https://stackoverflow.com/questions/tagged/yt-dlp)
+ [corbpie blog](https://write.corbpie.com/downloading-youtube-videos-and-playlists-with-yt-dlp/)

## Saving video as mp3 file

```bash
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=tJx7LbH_GwE'  -o '%(id)s.%(ext)s'
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH' -o '%(title)s.%(ext)s' 
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH' -o '%(channel_id)s/%(playlist_id)s/%(title)s.%(ext)s'
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH' -o '%(channel_id)s/%(playlist_id)s/%(id)s.%(title)s.%(ext)s'
```

## How to rename multiple files with Total Commander
+ [Helge Klein](https://helgeklein.com/blog/renaming-multiple-files-with-regular-expressions-in-total-commander/) for the rescue

## Alternative way of reaching similar result
+ [Fox Deploy](https://www.foxdeploy.com/blog/use-powershell-to-download-video-streams.html)

## Summary
If your device supports the communication over ethernet through the wire with the use of microUsb/usb-c or Iphone lightning to RJ45, then you are closer to get rid of wireless traffic, never the less up to my knowledge, not all devices can do this.
That's it.

Last edit: 2022.04.09
