---
layout: post
title: "Download Photos From Flickr Via API"
date: 2011-07-09 14:30
description: Example Python code that shows you how to make Flickr API Calls that will download all the sizes for a given photo_id.
comments: true
categories: python oauth flickr
---

In my [previous post](http://mkelsey.com/2011/07/03/Flickr-oAuth-Python-Example.html), I gave an example of how to authenticate a user against the Flickr API using their new OAuth authentication scheme.

I am going to expand on that post a bit, and show you how to download all the images for a given photo using the [flickr.photos.getSizes](http://www.flickr.com/services/api/explore/flickr.photos.getSizes) API call.  

If you followed my previous post, you should have a "token" file in your directory that contains the "oauth_token" and "oauth_token_secret" in it.  You should also have an "apikeys" file that contains your application "api_key" and "secret".  We will need some code to read these files to get the keys and secrets.

Here is what I came up with:

{% codeblock lang:python %}
class APIKeys(object):
    """Base class to read the APIKeys from file"""
    
    def __init__(self,filename='apikeys'):
        try:
            fp = open(filename)
        except IOError as e:
            if e.errno == errno.EACCESS:
                print "file does not exists"
            # Not a permission error.
            raise
        else:
            with fp:
            self.apikey = fp.readline().rstrip('\n')
            self.apisecret = fp.readline()
            fp.close()
				
class TokenKeys(object):
    """Base class to read the Access Tokens"""
    
    def __init__(self,filename='token'):
        try:
            fp = open(filename)
        except IOError as e:
            if e.errno == errno.EACCESS:
                print "file does not exists"
            # Not a permission error.
            raise
        else:
            with fp:
            self.token = fp.readline().rstrip('\n')
            self.secret = fp.readline()
            fp.close()

{% endcodeblock %}

The next task at hand was coming up with a base class that can encapsulate all the work needed in order to make subsequent Flickr API calls once you have your api key and token.  I named this class "FlickrApiMethod" and it sets up all the data needed to make Flickr API requests.  It has a "makeCall" method that will make the REST call, load the returned JSON content and return True if the call was successful, or False if it was not.  I plan on expanding on this a bit to support all the different response formats that Flickr supports, but for now it just supports JSON.  This class also has an no implementation "getParameters" method that allows subclasses to fill out additional parameters that are needed for a specialized API call.

Here is the code:

{% codeblock lang:python %}
class FlickrApiMethod(object):
    """Base class for Flickr API calls"""
    
    def __init__(self,nojsoncallback=True,format='json',parameters=None):
        apifile = APIKeys()
        tokenfile = TokenKeys()

        self.consumer = oauth.Consumer(key=apifile.apikey, secret=apifile.apisecret)
        self.token = oauth.Token(tokenfile.token, tokenfile.secret)
        
        if nojsoncallback:
            self.nojsoncallback = 1
        else:
            self.nojsoncallback = 0
        if not parameters:
            parameters = {}
            
        self.url = "http://api.flickr.com/services/rest"
        
        defaults = {
            'format':format,
            'nojsoncallback':self.nojsoncallback,
            'oauth_timestamp': str(int(time.time())),
            'oauth_nonce': oauth.generate_nonce(),
            'signature_method': "HMAC-SHA1",
            'oauth_token':self.token.key,
            'oauth_consumer_key':self.consumer.key,
        }
        
        defaults.update(parameters)
        self.parameters = defaults
        
    def makeCall(self):
        self.parameters.update(self.getParameters())
        req = oauth.Request(method="GET", url=self.url, parameters=self.parameters)
        req['oauth_signature'] = oauth.SignatureMethod_HMAC_SHA1().sign(req,self.consumer,self.token)
        h = httplib2.Http(".cache")
        resp, content = h.request(req.to_url(), "GET")
        self.content = content
        self.json = json.loads(content)
        
        if self.json["stat"] == "ok":
            return True
        else:
            return False
    
    def getParameters(self):
        raise NotImplementedError
{% endcodeblock %}


Next, I will write a class, that given a unique "photo_id" it will call the [flickr.photos.getSizes](http://www.flickr.com/services/api/explore/flickr.photos.getSizes) method and write out the files for all the sizes available.

{% codeblock lang:python %}
class FlickrPhotosGetSizes(FlickrApiMethod):
    name ='flickr.photos.getSizes'

    def __init__(self,nojsoncallback=True,format='json',parameters=None,photo_id=None):
        FlickrApiMethod.__init__(self,nojsoncallback,format,parameters)
        self.photo_id = photo_id

    def getParameters(self):
        p ={
            'method':'flickr.photos.getSizes',
            'photo_id':self.photo_id
        }
        return p
    
    def writePhotos(self):
        for o in self.json["sizes"]["size"]:
            opener = urllib2.build_opener()
            page = opener.open(o["source"])
            my_picture = page.read()
            filename = o["label"].replace(' ', '_') +"_" + self.photo_id + o["source"][-4:]
            print filename
            fout = open(filename,"wb")
            fout.write(my_picture)
            fout.close()
{% endcodeblock %}


Finally, a little bit of code to use these classes:

{% codeblock lang:python %}
print "Please enter a photo id:",
photoId = raw_input()
print "Fetching Photo"

photoSizes = FlickrPhotosGetSizes(photo_id = photoId)
if(photoSizes.makeCall()):
	print "API Call Success! Writing Photos to Disk"
	photoSizes.writePhotos()
else:
	print "API Call Failed"
	
{% endcodeblock %}

If all went well, you should now have files in your directory for all the different sizes of the photo you specified.

Feel free to email me with any questions or comments.  The git repo for this is located [here](https://github.com/kelsmj/FlickrOAuth).
