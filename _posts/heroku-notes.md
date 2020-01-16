Playing with Heroku's Go server. Perfect for small apps without many dependencies.



Procfile defines what commands to run.
It provides managed postgresql database, accessed by a given DB URL.
Can integrate with Docker easily
Access logs with `heroku logs --tail`
Scaling the app - you can add machines for it with `heroku ps:scale web=x`.
Modify configs on the fly - with .env files

# Dev environment
You can run your app locally with `heroku local`.

----------
How does it compare to a DigitalOcean server?

digital ocean is mostly built on your own. Automated with Docker. Heroku is actually a lot simpler and more functionalities.