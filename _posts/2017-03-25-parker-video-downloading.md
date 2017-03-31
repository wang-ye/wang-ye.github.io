Video Downloading With Celery and You-Get

Imagine you want to download the Youtube videos from your favourate channels automatically. How do you design it? [Parker](https://github.com/LiuRoy/parker)provides a good example.

The idea is as follows:
By using Celery beat, Parker crawls the channels periodically, and identify the new videos in the channel. Once new videos appear, Parker schedules another set of Celery tasks to download the video asynchronously, with the help of [you-get](https://github.com/soimort/you-get). The new videos metadata will be stored in mysql tables, while the downloaded videos are stored in local filesystem.

I will skip Parker walk through as they are pretty straightforward to understand, but instead talks about how you-get downloads the videos.
I will take Panda.tv as an example. If you want to crawl a panda.tv streaming video.

```
you-get http://www.panda.tv/387885
```

What it does:
A simple case. Given the input url, first identify the actual video URL through some way to get the hidden video url.
Then with request package, crawl the flv directly. Yes, the request package is the magic here!

For Youtube, it is a lot more complex. Feel free to check the source code.

What I like about this project:

1. Configurable system. The project puts all the configurations into the config directory. Inside the directory you can specify the logging format, the URL and port of various services such as Celery and Statsd, as well as the channels and playlists you want to monitor.
2. Monitoring effort
With the use of statsd and Grafana, Parker enjoys a very nice monitoring UI. You can go to the [README](https://github.com/LiuRoy/parker) to see the monitoring plots.
3. The use of Docker to simplify the system configuration. It recommends the use of a docker image to set up the Grafana monitoring with a single command.
How to quickly set up a production environment?
https://hub.docker.com/r/samuelebistoletti/docker-statsd-influxdb-grafana/
4. The great tool [you-get](https://github.com/soimort/you-get) to download any videos you want in command lines!

What can be done differently

A task retry mechnisum
More modualr code - extract the common pieces together.
I got to understand the power of docker. A docker image that configures everything and immediately runnable :)

