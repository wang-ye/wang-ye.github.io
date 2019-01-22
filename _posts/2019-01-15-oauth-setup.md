When integrating third-party services, authorization/authentication is usually the first thing to set up. Nowadays, most big companies such as Google and Microsoft recommend the OAuth flow. So what is it and how can we set it up correctly? In this doc, we will first review the different authorization strategies and why oauth is chosen. Then, I will use Bing OAuth as an example to illustrate the oAuth setup.

## Authorization Options
There are multiple ways for user to conduct authorization/authentication when using an API. This [post](https://nordicapis.com/3-common-methods-api-authentication-explained/) provides a good overview.

### Basic Authentication
The most basic form is HTTP Basic Authentication. A user would need to provide both username and password for auth purpose. This scheme is subject to man in the middle attacks, where a user can simply capture the login data and authenticate using the same credentials.

### API Keys
It is a common practice to provide API keys for data access. In this approach, each user is assigned an API key to future authentication purpose. It is relatively easy to implement at server side, and does not involve user credentials. Another benefit is the system can control the lifecycle of the API keys as well. The potential problem is the keys can be easily exposed when making API calls during network transmission.

### OAuth
OAuth is one of the latest additions to the authentication family. It can be quite complex. For a quick intro, please read the blog post [what the hack is oauth](https://developer.okta.com/blog/2017/06/21/what-the-heck-is-oauth), which covers the basics and all the major oauth flows. For our purpose( integrate with third-party APIs), we only need to understand one flow, *Authorization Code Flow*.

So what is this authorization code flow? Remember oauth is just a tool, and we use oauth for our authorization needs. During the auth code flow, our eventual goal is getting two tokens, an access token and  a refresh token. These two tokens will be passed to HTTP requests for all auth purposes. To get them, we need to explicitly authenticate on the web service hosting our data, and explicitly grant permissions to access the desired resource. Once the above steps are done, we will be able to retrieve the tokens used solely for accessing our agreed resources.

When making the request, we only use a very short-lived access token, which will be expired in hours or even minutes. Even if someone intercepts the HTTP requests, the token expires soon. Also, the token can only access limited resources. Both factors make the resources more secure.

## OAuth Setup
After some digging and experimenting, I finally set up the Bing OAuth and successfully updated our keyword bids using this API. In this section, I want to walk you through Bing's OAuth setup. It can also be applied to google oAuth and some other third-party integrations. Another thing to note is I am not talking about oauth from a web service perspective. Instead, all our usages are from making some API calls from some backend jobs, where I am both the owner of data and the API integration code.

How do we get the tokens? We need to first set up an app(client) and use it as a proxy for accessing the third-party data we need. Once we finish setting up the app(client), we can trigger an authorization request for the app, and authenticate ourself, and get the two tokens we need. More specifically,

1. Create the app. Usually, you need to create an native app in order to access your own data. This creates some frictions, but what can we do if the data provider is enforcing it? In this app, you would set the scopes and the permissions your app is allowed to do. Note that the permission you set here will affect your task. If you want to update the compaigns, but forget to include the compaign update permission, then your job will fail due to the permission errors.

2. Once your app is ready, the third-party would also provide a URL for you.
For Bing ads, the url is

```
https://login.live.com/oauth20_authorize.srf?client_id=CLIENT_ID&scope=bingads.manage&response_type=code&redirect_uri=REDIRECTURI&state=ClientState
```

You would need to specify the different fields (CLIENT_ID, REDIRECTURI, CLIENTSTATE), and go to this URL and authenticate. After the authentication, you will get the authorization code. This will be used in the next stage.
For Bing ads, it is something like

```
https://login.live.com/oauth20_desktop.srf?code=CodeGoesHere&state=ClientStateGoesHere.
```

The authorization code is in the URL.

3. With the authorization code, you can now exchange it for refresh and access tokens. For our case, another POST request is needed.

```
POST https://login.live.com/oauth20_token.srf HTTP/1.1
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Host: login.live.com
Content-Length: ContentLengthGoesHere

client_id=ClientIdGoesHere&scope=bingads.manage&code=CodeGoesHere&grant_type=authorization_code&redirect_uri=https%3A%2F%2Flogin.live.com%2Foauth20_desktop.srf
```

The response will contain the refresh_token and access_token. That's all we need for Bing API auth!

## Token Management
Usually, the access_token expires in hours, while the refresh_token could last for months or even years. When we request resources from third-party data providers, we can only use access_token.  How do we authenticate when the access_token expires? This is usually done via refresh token - we use refresh token to redeem access tokens, and then access the APIs with the new access token.

Another question is, if we need to run the app in our daily jobs, how can we keep it running without any user-initiated token updates? Fortunately, most oauth flows also provide ways to get updated refresh token. You can save them accordingly.

```python
def get_refresh_token():
    '''
    Returns a refresh token if stored locally.
    '''
    file=None
    try:
        file=open("refresh.txt")
        line=file.readline()
        file.close()
        return line if line else None
    except IOError:
        if file:
            file.close()
        return None

def save_refresh_token(oauth_tokens):
    '''
    Stores a refresh token locally. Be sure to save your refresh token securely.
    '''
    with open("refresh.txt","w+") as file:
        file.write(oauth_tokens.refresh_token)
        file.close()
    return None
```

## Summary
In this doc I briefly compared the some different auth options and walk through the set up of oAuth authentication using Bing API as an example. Hopefully this can be an useful reference when we set up oAuth authentication in the future.