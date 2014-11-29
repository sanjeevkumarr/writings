## Netflix Ice: Easily See Your AWS Costs ##

Netflix (https://github.com/netflix/ice)[Ice] is another of Netflix's OSS projects that parses your billing information from AWS to present some really useful graphs to break out where your high costs are.

Setting it up is pretty easy:

First, you need to sign up for Amazon's Detailed Billing (http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/detailed-billing-reports.html)[here].  This will place the reports in a S3 bucket of your choosing.

We'll go ahead and install it as a standalone application to make deployments easier, so install Tomcat on your server.  We've been using the standard Tomcat 7 install with no problems:
```
$ sudo apt-get install tomcat7
```

Ice is a Grails application, so you'll need to install Groovy and Grails in order to build a WAR file from the Ice source.  Netflix is harcoded to use Grails 2.2.1.
```
wget http://dl.bintray.com/groovy/maven/groovy-sdk-2.3.8.zip
unzip groovy-sdk-2.3.8.zip
wget http://dist.springframework.org.s3.amazonaws.com/release/GRAILS/grails-2.2.1.zip
unzip grails-2.2.1.zip
# place these wherever you want, you get the idea
export PATH=${PATH}:$HOME/grails-2.2.1/bin:$HOME/groovy-sdk-2.3.8/bin
```

Clone the Ice repository and build the project.
```
$ git clone git@github.com:Netflix/ice.git
$ cd ice
$ grails clean; grails war
```

Put the WAR file in Tomcat's webapps directory.  We'll also set up the temp directories used by Ice.
```
sudo cp target/ice.war /var/lib/tomcat7/webapps
sudo chown tomcat7:tomcat7 /var/lib/tomcat7/webapps/ice.war
sudo mkdir -p /mnt/ice_processor /mnt/ice_reader
sudo chown tomcat7:tomcat7 /mnt/ice_processor /mnt/ice_reader
```

We'll also configure Tomcat for that WAR.

```
$ cat /etc/tomcat7/Catalina/localhost/ice.xml
<Context className="org.apache.catalina.core.StandardContext"
         cachingAllowed="true"
         charsetMapperClass="org.apache.catalina.util.CharsetMapper"
         cookies="true" crossContext="false" displayName="Welcome to Ice"
         docBase="ice.war" path="/ice" privileged="false"
         reloadable="false" swallowOutput="false" useNaming="true"
         wrapperClass="org.apache.catalina.core.StandardWrapper">
</Context>
```

Configure Ice.  The basic configurations you'll need to put in /var/lib/tomcat7/webapps/ice/WEB-INF/classes/ice.properties are:

```
# milliseconds since epoch to start billing at (if you wanted to start later)
ice.startmillis=1370044800000
# used in the display
ice.companyName=YourCompany
# the top-level S3 bucket that you specified when setting up AWS Detailed Billing above 
ice.billing_s3bucketname=traceview-billing
# the top-level S3 bucket used for Ice's flat file DBs
ice.work_s3bucketname=netflix-ice-workdir
# the temporary directories on your server used for Ice's flat file DBs, also mirrored above, give tomcat +rwx access
ice.processor.localDir=/mnt/ice_processor
ice.reader.localDir=/mnt/ice_reader
# your Amazon account number
ice.account.YourCompany=12345678
```

You'll also need to set these so Ice can access S3 to get the billing data.
```
ice.s3AccessKeyId=<accessKeyId>
ice.s3SecretKey=<secretKey>
ice.s3AccessToken=<accessToken>
```
At Appneta, we've used a quick IAM role instead, and given our server that instance profile.
```
{
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:List*",
        "s3:Put*",
        "s3:Get*",
        "s3:DeleteObject"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

Restart Tomcat afterwards.
