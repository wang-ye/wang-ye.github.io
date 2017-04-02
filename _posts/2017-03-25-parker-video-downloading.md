---
layout: post
title:  "Video Downloading With Celery and You-Get"
date:   2017-03-30 19:59:03 -0800
---

Imagine you want to download some Youtube videos from your favorite channels whenever new videos are added. How do you achieve this? [Parker](https://github.com/LiuRoy/parker) provides a very interesting example.

The idea is as follows:
User first configures the channels they want to monitor and download.
Then, by using Celery beat, Parker crawls the channels periodically, and identify the new videos in the channel. Once new videos appear, Parker schedules another set of Celery tasks to download the video asynchronously, with the help of [you-get](https://github.com/soimort/you-get). The downloaded videos are stored in local filesystem.

I will skip the code walk through as they are pretty straightforward to understand, but instead discuss how you-get downloads the videos.
Panda.tv is one of my favorite video streaming website. To crawl a panda.tv streaming video, you can simply call

```
you-get http://www.panda.tv/387885
```

What it does:
Given the input url http://www.panda.tv/387885, you-get first identifies the actual video steaming URL with some nice hacks. It turns out the streaming video always has a flv format. With request package, you-get crawls the streaming content directly. Yes, the request package is the real magic here!
For some other video website, the crawling process can be a lot complex, i.e., Youtube. Feel free to check the source code.

Now go back to Parker. Here is what I like most about this project:

1. Configurable system. The project puts all the configurations into the config directory. Inside that directory you can specify the logging format, the URL and port of the Celery, Statsd and monitoring services, as well as the channels you want to monitor.
2. Monitoring-first design
With the use of statsd and Grafana, Parker enjoys a very nice monitoring UI. You can go to its [README](https://github.com/LiuRoy/parker) to see the actual monitoring plots.
3. The use of Docker to simplify the system configuration. Parker recommends a docker image to set up Grafana monitoring with a single command! Here is [the link](https://hub.docker.com/r/samuelebistoletti/docker-statsd-influxdb-grafana/) to the docker monitoring image.
4. The great tool [you-get](https://github.com/soimort/you-get) to download any videos you want in command lines!

The takeaway - I am deeply impressed by the power of Docker and plan to use it more in the future.
