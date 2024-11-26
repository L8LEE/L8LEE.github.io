---
layout: post
title:  "Making a webp Timelapse Animation from ipcam rtsp Captured Images using ffmpeg"
date:   2024-11-25
categories: jekyll update
---

I went through several iterations of trying to create an animation that can be displayed on github pages. Github pages does not support embedded youtube videos, so getting the animation to display is a bit of a challenge given storage space limitations on both github pages and most image hosting sites. I finally arrived at this solution :

![one_day](https://i.ibb.co/xXSr3T6/oneday.webp)

So how was that made? The first step is capturing the images. Those images that make up that animation were captured from a $17 ipcam.

![mycam](https://i.ibb.co/9N5jX4R/mycam.webp)

Here is how I currently have the camera mounted.

![cam_mount](https://i.ibb.co/92hrq96/ipcam-mount.webp)

It's simply mounted under some old scrap barn tin to keep it out of direct exposure to the rain. 

Once the camera is mounted and connected to my router, I can then access the rtsp stream through my local account. I set up a shell script and a cronjob to capture an image once every 5 minutes. Here's the code.

```bash
#!/bin/bash
# LOCATION is where you want to store your captures
LOCATION=/home/lee/Pictures/back_yard
# FILENAME is unix time
FILENAME=$(date +%s).jpg
# your RTSP address from your camera settings will be different
NET_LOC=rtsp://RQgu86FV:UtY2GaA5EjO3ujSP@192.168.1.48:554/live/ch0
# finally the ffmpeg command that captures the images
ffmpeg -y -loglevel fatal -rtsp_transport tcp -i $NET_LOC -frames:v 1 $LOCATION/$FILENAME
```

The above bash script will capture one jpg image from the camera's rtsp stream. Then set up the cronjob to run at your desired interval. This is for once every 5 minutes.

```bash
# m h  dom mon dow   command
*/5 * * * * /home/lee/.local/bin/back_yard_capture.sh
```

Now that the images have been captured, it's time to build the animation. GIF animations have been a popular choice for posting animated images to image hosting sites. But for my use case, the file sizes were just way too large and I was exceeding the upload limit for my chosen host, imgbb. So I had to do some crunching. I tried avif, but the processing was taking way, way too long and the results weren't that much better than webp. So I decided to go with webp.

There are several steps involved to get the final animation. I'll go through each of those below. First the images the above script captures from my camera are 2304x1296 jpg's and they range in size from 40kb at night to 430kb in the day. The first gif I made was from 575 captured images over two days. That led to a gif that was 484.2 MB, way too large for my 32 MB upload limit. I tried making some webm videos, and even they were still too large at 116.3 MB.

I did several more tests and finally arrived at the following workflow. First downscale the images to 720 width while maintaining the original aspect ratio. This greatly reduces the size of each image from the start. Here is the script I made that downscales and resizes all the captured images in a particular folder.

```bash
#!/bin/bash

x=1
for FILE in *.jpg
do
 COUNTER=$(printf %03d $x)
 OUTFOLDER=downscale
 NEWFILE=$COUNTER.webp
 ffmpeg -i $FILE -c:v libwebp -quality 50 -vf scale=720:-1 $OUTFOLDER/$NEWFILE
 x=$(($x+1))
done
```
That script will take a 405 kb jpg file and get it down to a 66.7 kb webp file.

Next, I decided to make each animation exactly one day long. So 12 files per hour times 24 hours is 288 consecutive images.

Then it's just one more line of code to make the webp animated image from the 288 chosen files.

```bash
ffmpeg -start_number 001 -framerate 6 -i %03d.webp output.webp
```

Notice the use of the formatting strings (%03d) in the above codes. This is to ensure the sequences are in order and numbered 001,002,003,004....286,287,288 etc. ffmpeg works much better when the images are sequenced like that.

After all is said and done, the resulting webm animation for the 24 hour period was only 12.5 MB. So there is quite a bit of room to make some improvements. First I could use (-vf scale=1080:-1) in place of (-vf scale=720:-1) to increase the resolution. And I could also use (-quality 75) to raise the quality of the conversion. Maybe then the pixelation seen at night could be reduced. 

