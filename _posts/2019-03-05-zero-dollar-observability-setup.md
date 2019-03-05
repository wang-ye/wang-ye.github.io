Capturing errors timely in your software system is crucial. There are many excellent enterprise-level tools available for this purpose. However, most of them would charge you some extra money even you just need a very basic version. In this post, I will discuss how I set up small project observability with zero dollar cost, by utilizing the free or open source tools available. This setup has been running well for me. I hope more people could benefit from this setting.

## What are the goals?
In short, we want to achieve the following:
1. Low to zero cost for the observability
2. (near-)realtime communication with developers
3. Exception/Error capturing
4. Visualization of metric trends, better yet to have timeseries based alerting

## How?
There are open source tools and freemium products that provides a limited set of features for free users. However, these features should be enough for a small-scale project. For our observability use case, I use the combination of Slack, Sentry, Sendgrid, Datadog and Grapana.

### Slack
Slack plays a central rule in the whole observability effort. It is first a messenger tool where every team member can communicate with each other, and more importantly, it is also key to communicate the alerts and errors timely to the developers. By integrating Slack with your App, you can send error log messages and important status updates to a slack channel immediately.

### Email
I mainly use Slack messages to get real-time updates, and rarely use emails. For those interested in setting up emails, try Sendgrid. You can send at most 100 emails per day even in the free plans. A even better way, in my opinion, is setting up a slack integration for sending emails when an important message arrives. By doing this, your app only needs to communicate with Slack.

### Exception Catching
Sentry is used for capturing the exceptions. By integrating with Sentry, every exception is captured. Previously I wrote a article regarding the exception catching mechanism. Take a look if you are interested.

### Timeseries Metrics and Rule-based Alerts
Datadog is the de facto tool for collecting metrics and stats. It also provides a free plan with only one day worth of data, and no alerting possible under the free plan. However, you can further redirects the stats to your own grapana server for better alerting purpose.

## Summary
For small-scale apps, you can set up your own observability toolkit with zero cost, although the features freemium tools provided are also a bit limited. Also, once your app becomes bigger, you can always switch to the paid plans and get more useful features with ten to twenty dollars each month. Hope this article is helpful during your hacking!