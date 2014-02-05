## Dr. Ice, Or How I Learned About S3 List ##

AppNeta has used a lot of open source libraries and programs in building and running our architecture.  One utility in general that's provided us with an easy way to slice up and investigate our AWS spending is the awesome [Ice](https://github.com/Netflix/ice/).  Instead of having to do manual tabulation based on the monthly billing email from Aamzon, we can easily break down and graph our bill by hour, week or month.  By tagging our resources, we can even group by environment, role or any other category we want, making it easy to see how much it costs to run our production environment versus our staging environment and other expirements.

It also allows us to easily spot aberrations in our usage patterns which cost AppNeta money.  One of those was a somewhat high cost in our S3 bill, where we noticed that we were performing a _lot_ of S3 List operations, as noticed by our engineering team.

We use Python pretty heavily, and are big fans of [boto](http://boto.readthedocs.org/en/latest/).  When retrieving and storing objects in S3, we usually execute code similar to:

```python
 def _get_bucket(self, object_type):
      """ gets or creates bucket for object type """
      bucket_name = '%s-%s' % (self.bucket_prefix, object_type)
      try:
          bucket = self.s3.get_bucket(bucket_name)
      except S3ResponseError, e:
          if e.status == 404: # Bucket not found, first time using bucket.
              bucket = self.s3.create_bucket(bucket_name)
```

There's nothing really out of the ordinary here, but we usually know the bucket name when using this operation, and we make sure to create the bucket first before placing code in production that needs a new bucket.  Digging further into the boto code, we [found](https://github.com/boto/boto/blob/master/boto/s3/connection.py#L412):

```python
def get_bucket(self, bucket_name, validate=True, headers=None):
    """
    Retrieves a bucket by name.

    If the bucket does not exist, an ``S3ResponseError`` will be raised. If
    you are unsure if the bucket exists or not, you can use the
    ``S3Connection.lookup`` method, which will either return a valid bucket
    or ``None``.

    :type bucket_name: string
    :param bucket_name: The name of the bucket

    :type headers: dict
    :param headers: Additional headers to pass along with the request to
        AWS.

    :type validate: boolean
    :param validate: If ``True``, it will try to fetch all keys within the
        given bucket. (Default: ``True``)
    """
    bucket = self.bucket_class(self, bucket_name)
    if validate:
        bucket.get_all_keys(headers, maxkeys=0)
    return bucket
```

The important parameter to notice here is the default of ```validate=True```, which is also listed as the first example of getting a bucket in the boto [docs](http://boto.readthedocs.org/en/latest/s3_tut.html).  This causes the code to call get_all_keys, which calls S3's LIST command with maxkeys=0 to verify the bucket exists ([here](https://github.com/boto/boto/blob/master/boto/s3/bucket.py#L369)) and returns a [ListBucketResult](http://docs.aws.amazon.com/AmazonS3/latest/API/RESTBucketGET.html).

Now, normally S3 is really, _really_ [cheap](http://aws.amazon.com/s3/pricing/).  But LISTs are _over 12 times_ more expsensive than GETs.

| PUT, COPY, POST, or LIST Requests | $0.005 per 1,000 requests |
| GET and all other Requests | $0.004 per 10,000 requests |

At scale, even $0.0005 per 1000 requests adds up.

Since we almost always know the bucket we're going to be accessing or writing to, we have no need to perform an extra List, so we can modify our code to:

```python
          bucket = self.s3.get_bucket(bucket_name, validate=False)
```

Our usage is now a lot less, which translates to a lower cost and a happier executive team :) 

![](https://raw.github.com/jessedavis/writings/master/images/s3-list-fixed.png)

For fun, our dev team decided to see how pervasive the use of the default validate parameters might be available on Github:

![](https://raw.github.com/jessedavis/writings/master/images/github-get_bucket_results.png)

_Ouch._
