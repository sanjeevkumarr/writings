## Netflix Ice: Easily See Your AWS Costs ##

Netflix [Ice](https://github.com/netflix/ice) is another of Netflix's OSS projects that parses your billing information from AWS to present some really useful graphs to break out where your high costs are.

Setting it up is pretty easy.  First, you need to sign up for Amazon's Detailed Billing [here](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/detailed-billing-reports.html).  This will place the reports in a S3 bucket of your choosing.  Make sure you pick the option to export your resource tags as well.

You can use Netflix's [install](https://github.com/Netflix/ice/blob/master/install.sh) script to set up the project as a Grails application, and there's even a community Chef cookbook to install it.  At AppNeta we've installed it in one of our Tomcat servers to make deployments easier.  We've been using the standard Tomcat 7 install with no problems:
```
$ sudo apt-get install tomcat7
```

Ice is a Grails application, so you'll need to install Groovy and Grails in order to build a WAR file from the Ice source.  Netflix is hardcoded to use Grails 2.2.1.
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
# the list of tags you exported in your billing CSV to use in application groups
ice.customTags=user:Role
```

You'll also need to set these so Ice can access S3 to get the billing data.
```
ice.s3AccessKeyId=<accessKeyId>
ice.s3SecretKey=<secretKey>
ice.s3AccessToken=<accessToken>
```
We've used a quick, albeit permissive, IAM role instead and given our server that instance profile.
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

Restart Tomcat afterwards, and you should be able to access Ice at http://yourserver:8080/ice .  If you don't see any data yet, check your billing S3 bucket to see if any reports have been delivered.  Amazon generates them several times a day, so you'll eventually see CSVs there that Ice will periodically download.

The basic screen will show your cost breakdown by AWS product type:

![](https://raw.github.com/jessedavis/writings/master/images/basic_resource_breakdown.png)

If you tag your instances, EBS volumes and other resources, you can define application groups to group them together.

![](https://raw.github.com/jessedavis/writings/master/images/application_group_menu.png)

After you've defined your application groups, you get a nice default view that shows the cost breakdown per group, which is pretty useful to use in reports for dev vs. production environment cost.

![](https://raw.github.com/jessedavis/writings/master/images/application_group_list.png)

You can then click on the application group and filter by Product type to see your server cost per group.

![](https://raw.github.com/jessedavis/writings/master/images/resource_group_instance_cost.png)

The little amount of time it took us to install and configure Ice has already paid off.  We used it to spot an boto design decision that was costing us quite a bit of money (see [here](http://www.appneta.com/blog/s3-list-get-bucket-default/) for the lowdown).  Our pull request even made it back into [boto](http://boto.readthedocs.org/en/latest/releasenotes/v2.25.0.html)!
