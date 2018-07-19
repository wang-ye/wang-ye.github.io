In this post I want to share a personal project I just completed. The project, or *Yup!*, crawls and downloads videos that users like, and then uploads them to Youtube channels.

# Key Components
1. Video Metadata crawler - Users specify the keywords to crawl, and then the crawler would crawls the metadata and stores them in Postgres DB.
2. Video Selector - Users can then access the video metadata, and specify whether you want to download the videos or not.
3. Video Downloader - Downloads the videos accordingly, and stores the results in filesystem.
4. Video Uploader - Uploads the videos to Youtube
5. Yup! uses Flask for user interfaces and download/upload status check, and celery for job scheduling. 

# How to quickly Prototype?
* The key for fast prototyping is reusing as much as possible :)
* Lay down key interfaces first - with clearly defined input/output
* For experimental code, use juypter to quickly prototype
* Docker container will be very useful if your service involves multiple infra components (db, redis, load balancer, web server ...)
* Optimize for simplicity

# Tech Debt
monitor job status in realtime
Effectively work with the data queries inside container
setup logging for project properly
Understanding more about docker private network
Write Tests ..
