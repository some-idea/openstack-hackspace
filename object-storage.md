# Object storage

OpenStack Object Storage allows you to store files. Here we'll use Rackspace Cloud Files, which is built on top of OpenStack Object Storage. 

## Upload a file

Here we're going to upload a file into the "hackspace" container. We'll use an OpenStack logo as our file:

![image](http://www.openstack.org/assets/openstack-logo/openstack-cloud-software-vertical-small.png)


Download the file [openstack-cloud-software-vertical-small.png] to your local machine:

	$ wget http://www.openstack.org/assets/openstack-logo/openstack-cloud-software-vertical-small.png 
	
Now, upload this file to object store into a new container called "hackspace":

    $ swift upload hackspace openstack-cloud-software-vertical-small.png
    
The hackspace container should now appear:

    $ swift list
    hackspace

The contents of the container should be the file:

	$ swift list hackspace
	openstack-cloud-software-vertical-small.png

[openstack-cloud-software-vertical-small.png]: http://www.openstack.org/assets/openstack-logo/openstack-cloud-software-vertical-small.png

## Download a file

Download the file from Openstack Object Storage to your local machine, giving it a different name (logo.png):

    $ swift download hackspace openstack-cloud-software-vertical-small.png --output logo.png
    

## Make the file accessible via temporary url

We'll generate a temporary url to allow others to access the `hackspace/openstack-cloud-softare-vertical-small.png` file. 

First, we need to set a secret key on the account. Here we'll use `super-secret-key` as the key:

    $ swift post -m "Temp-URL-Key:super-secret-key"


Next, we need to determine the full url of the object. For that, we need to get the storage url. We need to query the REST API directly for that:

    $ curl -i -s $ST_AUTH -H X-Storage-User:$ST_USER -H X-Storage-Pass:$ST_KEY | grep X-Storage-Url
    X-Storage-Url: https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013
    
In our example here, the storage URL is:
 * `https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013`
 
The full URL for the object is:
 * `https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-softare-vertical-small.png`

We can ensure this url works by making a HEAD request using an authentication token. First, retrieve an auth token:

	$ curl -i -s $ST_AUTH -H X-Storage-User:$ST_USER -H X-Storage-Pass:$ST_KEY  | grep '^X-Auth-Token'
	X-Auth-Token: fac73d71321c4fdbac95ed8a183ff384	
Next, make a HEAD request against the object url, passing the auth token, and specifying that curl print out headers (`-i`):
	
	$ curl -i -X HEAD -H X-Auth-Token:fac73d71321c4fdbac95ed8a183ff384 https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-software-vertical-small.png
	HTTP/1.1 200 OK
	Content-Length: 6685
	Content-Type: image/png
	Accept-Ranges: bytes
	Last-Modified: Sat, 02 Nov 2013 02:41:36 GMT
	Etag: 9ca71ae6465e5d4228e5c1e2f354c80c
	X-Timestamp: 1383360096.69969
	X-Object-Meta-Mtime: 1313113538.000000
	X-Trans-Id: txc7d304c99e6447ec85cca-00527470f2iad3
	Date: Sat, 02 Nov 2013 03:26:42 GMT
	
	
Let's generate a temporary url that's good until Dec. 31, 2013 at 23:59:59. 

We need to write some Python code for this, we can do it directly in the Python interpreter. Note that we need the url path for generating the url. In the above case, it's `/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-software-vertical-small.png`, but your account name will be different.


	import hmac
	from hashlib import sha1
	import time
	from datetime import datetime
	d = datetime(2013, 12, 31, 23, 59, 59)
	expires = int(time.mktime(d.timetuple()))
	host = "storage101.iad3.clouddrive.com"
	# Note: your account will be different here
	account = "MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013"
	path = "/v1/{0}/hackspace/openstack-cloud-software-vertical-small.png".format(account)
	key = 'super-secret-key'
	hmac_body = 'GET\n{0}\n{1}'.format(expires, path)
	sig = hmac.new(key, hmac_body, sha1).hexdigest()
	s = 'https://{host}{path}?temp_url_sig={sig}&temp_url_expires={expires}'
	url = s.format(host=host, path=path, sig=sig, expires=expires)
	print(url)
	
	
The URL should look something like:


`https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-software-vertical-small.png?temp_url_sig=ad1cad4ee2856c0a566636678e5ce023e84a577a&temp_url_expires=1388552399`

This special url can be used to read the file until the expiry date.


## Make a container readable by all

Set the metadata on the hackspace container such that anybody can access the contents. Before:

	$ curl -i -X HEAD https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-software-vertical-small.png
	HTTP/1.1 401 Unauthorized
	Content-Type: text/html; charset=UTF-8
	Content-Length: 0
	X-Trans-Id: txa44484f152424b21b013c-00527479afiad3
	Date: Sat, 02 Nov 2013 04:03:59 GMT

Setting the access control list to allow read access and container listing from any referrer:

    $ swift post hackspace --read-acl '.r:*,.rlistings' 

Verify it's been set:

	$ swift stat hackspace
	  Account: MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013
	Container: hackspace
	  Objects: 1
	    Bytes: 6685
	 Read ACL: .r:*
	Write ACL:
	  Sync To:
	 Sync Key:
	Accept-Ranges: bytes
	X-Timestamp: 1383359834.10868
	X-Trans-Id: tx12ab73e20369465fbc17f-00527479cbiad3
	Content-Type: text/plain; charset=utf-8


Now users should be able to access the url: https://storage101.iad3.clouddrive.com/v1/MossoCloudFS_4911155b-84c6-448b-b0f3-7db8f4887013/hackspace/openstack-cloud-software-vertical-small.png