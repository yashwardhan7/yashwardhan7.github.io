---
layout: post
title:  "ImageMagick heavy CPU load issue and RaspberryPi freeze"
date:   2016-06-19
categories: imagemagick raspberrypi timelapse camera
excerpt_separator: <!--more-->
---
I've been working on a python script to create a timelapse capture using RaspberryPi and the Pi(NoIR) camera.
I wanted to generate thumbnails for the captured images and decided to use ImageMagick's convert program.
But I started getting these seemingly random freezes and often even requiring a hard reboot.
<!--more-->
After some searching, I came across [this post from 2012 that describes an OpenMP bug in ImageMagick and possible workarounds](http://www.azanweb.com/en/high-cpu-load-when-converting-images-with-imagemagick/).
The bug, as reported, happens for the OpenMP build of ImageMagick on a multi-core/proc machine. My setup met both these criteria.

Two possible workarounds were suggested:

1. Recompile ImageMagick with OpenMP feature disabled.
2. Run the required program after setting the environment variable, MAGICK_THREAD_LIMIT=1

I quickly tested the environment variable based workaround and it worked. I no longer get the random freezes.
Here's the Python snippet I used:
{% highlight python %}
  cmd = 'convert file.jpg -resize 266x200 -quality 50 thumb.jpg'
  custom_env = os.environ.copy()
  custom_env['MAGICK_THREAD_LIMIT'] = '1'
  proc = subprocess.Popen(cmd.split(), shell=False, env=custom_env)
  proc.wait()
{% endhighlight %}
