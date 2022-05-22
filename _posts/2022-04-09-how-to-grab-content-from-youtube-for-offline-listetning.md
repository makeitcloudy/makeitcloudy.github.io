---
layout: post
title: "How to grab content from youtube for offline listening"
permalink: "/how-to-grab-content-from-youtube-for-offline-listening/"
subtitle: "You don't need to have premium plan"
cover-img: /assets/img/path.jpg
thumbnail-img: /assets/img/thumb.png
share-img: /assets/img/path.jpg
tags: [Audio ,Misc]
categories: [Audio ,Misc]
---

## Prerequisities

Download and install [ffmpeg](https://ffmpeg.org/)

## Howto

This can be used for youtube, but also for other network streams, provided you know the path of the media stream, which can quite easily be identified with developer tools comming from the web browser.

* [youtube-dl](https://github.com/ytdl-org/youtube-dl)
* [yt-dlp](https://github.com/yt-dlp/yt-dlp)

## Where to find some usefull details

+ Github and [stackoverflow](https://stackoverflow.com/questions/tagged/yt-dlp)
+ [corbpie blog](https://write.corbpie.com/downloading-youtube-videos-and-playlists-with-yt-dlp/)

## Saving video as mp3 file

```bash
yt-dlp -f 'ba' -x --audio-format mp3 https://www.youtube.com/watch?v=tJx7LbH_GwE  -o '%(id)s.%(ext)s'
yt-dlp -f 'ba' -x --audio-format mp3 https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH -o '%(title)s.%(ext)s' 
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH' -o '%(channel_id)s/%(playlist_id)s/%(title)s.%(ext)s'
yt-dlp -f 'ba' -x --audio-format mp3 'https://www.youtube.com/watch?v=SWnfGJ36gpQ&list=PLuF78wm0RiGbo4vZ2eqgicrKkZDq3BCkH' -o '%(channel_id)s/%(playlist_id)s/%(id)s.%(title)s.%(ext)s'
```

## How to rename multiple files with total commander
+ [Helge Klein](https://helgeklein.com/blog/renaming-multiple-files-with-regular-expressions-in-total-commander/) for the rescue

## Summary

That's it.
