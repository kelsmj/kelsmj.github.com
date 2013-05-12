---
layout: post
title: "OAuth User Authentication for Flickr in Python"
date: 2011-07-03 14:30
description: Example Python code that shows you how to authenticate using the OAuth authentication scheme for Flickr.
comments: true
categories: python oauth flickr
---

I always wanted to write an application that will use the Flickr API to pull all my photos from Flickr and store them locally, so I can further back them up via [Crashplan](http://www.crasphlan.com).

I decided to write this app in python, a language which I have no professional experience in, and have only recently dabbled with.  I figured it would be a good learning experience.  We will see!

The first thing that needs to be accomplished, in order to make calls to the Flickr API, is authenticate a user.  Flickr just recently added support for using OAuth to authenticate a user [Using OAuth with Flickr](http://www.flickr.com/services/api/auth.oauth.html) and they will be removing their old authentication API in the future.  The focus of this post will outline the steps I took in order to successfully use OAuth to authenticate against my application.

The code below has two prerequisites in order to work correctly:

1. A text file called "apikeys" in the root directory containing the api_key (consumer_key) and secret for your application.  You get the key and secret by creating a new Flickr Application associated with your account.  Go to "The App Garden":http://www.flickr.com/services/apps/create/apply/ to apply for one.  Each entry in this file needs to be on its own line with the api_key coming first.
2. Downloading and installing "python-oauth2":https://github.com/simplegeo/python-oauth2

To setup python-oauth2:

{% codeblock %}
~/$ git clone https://github.com/simplegeo/python-oauth2.git
~/$ ./setup.py install
{% endcodeblock %}

The first thing to understand about using OAuth with Flickr, is every request must be signed using the HMAC-SHA1 signature encryption.  You do this by building a Request object via the python-oauth2 Request method, passing in the url you want to make the request to and the parameters you want to send.  You then call the SignatureMethod_HMAC_SHA1().sign method and add the resulting signature to the Request.

The begin the authentication procedure, you first need to create a request that will get a request token.  The request will be made to http://www.flickr.com/services/oauth/request_token with the following paramaters:
* oauth_timestamp
* oauth_signature_method
* oauth_version
* oauth_callback
* oauth_nonce
* oauth_consumer_key
* oauth_signature

The code to perform this operation:

{% codeblock lang:python %}
url = "http://www.flickr.com/services/oauth/request_token"

# Set the base oauth_* parameters along with any other parameters required
# for the API call.
params = {
	'oauth_timestamp': str(int(time.time())),
	'oauth_signature_method':"HMAC-SHA1",
	'oauth_version': "1.0",
	'oauth_callback': "http://www.mkelsey.com",
	'oauth_nonce': oauth.generate_nonce(),
	'oauth_consumer_key': keys.apikey
}

# Setup the Consumer with the api_keys given by the provider
consumer = oauth.Consumer(key=keys.apikey, secret=keys.apisecret)

# Create our request. Change method, etc. accordingly.
req = oauth.Request(method="GET", url=url, parameters=params)

# Create the signature
signature = oauth.SignatureMethod_HMAC_SHA1().sign(req,consumer,None)

# Add the Signature to the request
req['oauth_signature'] = signature

# Make the request to get the oauth_token and the oauth_token_secret
# I had to directly use the httplib2 here, instead of the oauth library.
h = httplib2.Http(".cache")
resp, content = h.request(req.to_url(), "GET")
{% endcodeblock %}

If all is successful, Flickr include a _oauth_token_ and a _oauth_token_secret_ in the Response.  You can now store those tokens off and prompt a user to go authorize your application.  They will need to go to http://www.flickr.com/services/oauth/authorize with the _oauth_token_ appended as a querystring parameter along with the perms parameter indicating if your application will need read,write and delete privileges.  Once the user authorizes your application, Flickr will redirect to the _oauth_callback_ url specified above with a _oauth_verifier_ querystring parameter that you can then use in the third and final step of oAuth authentication.

The code to perform this operation:

{% codeblock lang:python %}
authorize_url = "http://www.flickr.com/services/oauth/authorize"

#parse the content
request_token = dict(urlparse.parse_qsl(content))

print "Request Token:"
print "    - oauth_token        = %s" % request_token['oauth_token']
print "    - oauth_token_secret = %s" % request_token['oauth_token_secret']
print

# Create the token object with returned oauth_token and oauth_token_secret
token = oauth.Token(request_token['oauth_token'],
	request_token['oauth_token_secret'])

# You need to authorize this app via your browser.
print "Go to the following link in your browser:"
print "%s?oauth_token=%s&perms=read" %
	(authorize_url, request_token['oauth_token'])
print

# Once you get the verified pin, input it
accepted = 'n'
while accepted.lower() == 'n':
    accepted = raw_input('Have you authorized me? (y/n) ')
oauth_verifier = raw_input('What is the PIN? ')

#set the oauth_verifier token
token.set_verifier(oauth_verifier)
{% endcodeblock %}

The third and final step in authenticating is exchanging the Request Token for an Access Token.  The Access Token is something you will store off for that user and you will be able to make Flickr API calls using the Access Token.

{% codeblock lang:python %}
# url to get access token
access_token_url = "http://www.flickr.com/services/oauth/access_token"

# Now you need to exchange your Request Token for an Access Token
# Set the base oauth_* parameters along with any other parameters required
# for the API call.
access_token_parms = {
	'oauth_consumer_key': keys.apikey,
	'oauth_nonce': oauth.generate_nonce(),
	'oauth_signature_method':"HMAC-SHA1",
	'oauth_timestamp': str(int(time.time())),
	'oauth_token':request_token['oauth_token'],
	'oauth_verifier' : oauth_verifier
}

#setup request
req = oauth.Request(method="GET", url=access_token_url,
	parameters=access_token_parms)

#create the signature
signature = oauth.SignatureMethod_HMAC_SHA1().sign(req,consumer,token)

# assign the signature to the request
req['oauth_signature'] = signature

#make the request
h = httplib2.Http(".cache")
resp, content = h.request(req.to_url(), "GET")

#parse the response
access_token_resp = dict(urlparse.parse_qsl(content))

#write out a file with the oauth_token and oauth_token_secret
with open('token', 'w') as f:
	f.write(access_token_resp['oauth_token'] + '\n')
	f.write(access_token_resp['oauth_token_secret'])
f.closed
{% endcodeblock %}

If all the requests succeeded you should have a token file in your directory with the _oauth_token_ and the _oauth_token_secret_.  You can use these tokens to make subsequent requests to the Flickr API.  In my next post, I will illustrate how to do this.

Feel free to email me with any questions or comments.  The git repo for this is located [here](https://github.com/kelsmj/FlickrOAuth)