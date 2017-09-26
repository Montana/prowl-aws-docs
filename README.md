![documentations](http://getprowl.com/assets/images/documentation1.png)

## Amazon Web Services for Prowl 

This is documentaiton in regards to how we are using Prowl with AWS.

## Amazon QuickSight

QuickSight is from the AWS suite that is simply a tool to visually build ad-hoc queries. As of now, there is no API for it so that we can automatically generate dashboards and there is no built in way to run statistical testing. For this, we can use Plotly, or something similar.

## Vagrant 

In the private Prowl repo's there are some VMWare Fusion license keys "add-ons" for Vagrant, worth about $70 a piece, still available if anyone involved in Prowl needs a key. We will be using precompiled Gentoo boxes for most testing via Vagrant. https://gentoo.org/

You can access this box, that's maintained by myself (Montana) via 

<pre>vagrant init montana/gentoo64</pre>
<pre>vagrant up</pre>
<pre>vagrant ssh</pre> 

If there is a port collision, I've already released a patch for this box, so port collisions should be fixed, but in the case that there is run 

<pre>vagrant halt</pre> 

Then reenter those above commands. 

## OpenRefine with Conciliator 

Some of the Prowl team have talked about using OpenRefine, about cleansing our data we receive in CSV form. Which as of now we have a whole lot. We had initially looked at CrowdFlower for this, but decided to see if we couldn't use something a bit more cost efficient, and open source. We would be using OpenRefine (2.5) which then was called Google Refine, the reason for this, is stability, along with an addon called "Conciliator". 

## Conciliator | AWS 

We have to make sure we have Maven installed, once installed, we can start cleaning data -- and sending this data to AWS. The flowchart below, describes this process. This flowchart was created by Garrett Loh, Prowl's co-founder, and business development. 

<p align="center">
  <img src="http://www.getprowl.com/assets/images/flowchart.png" alt="Flowchart"/>
</p>

## The process

Each Raw data dump should be in a folder named by the client, for example `EPR` or `HAUS COFFEE`.

We need the File name by upload date/time MMDDYYYY_HHMM (pref that format, but another will do) 
The data cleaned with Google Refine in an AWS S3 bucket called something like `prowl-clean`.

With this data we can connect S3 to AWS Glue. So now we know how the Data structure is setup like, the PoC is as follows in this chart.

![AWS](http://www.getprowl.com/assets/images/flower.png)

## Automation 

I would use Jenkins in any other case, but to stay within the suite, automation here calls for AWS Lambda. Given the notion our employees have the AWS CLI setup, you can run 

<pre>aws lambda list-functions --profile adminuser</pre> 

This will try and reach Lambda, if it fails, it will print out an error message. Prowl already has a custom `s3-get-object` and a `django-storages` via Zappa. 

<p align="center">
  <img src="http://i.imgur.com/f1PJxCQ.gif" alt="Zappa Prowl Demo Gif"/>
</p>


## Using the Zappa environment 

Okay, now that you're connected to Lambda, and you have hopefully grabbed our custom `django-storages` in the private repo, make sure Zappa has ALL of it's dependencies installed before you start using it, of course pip will fetch whatever is in the requirements.txt file, so let's start the Zappa shell, get the virtual enviornment going (you can use Docker alternatively), and install the requirements

<pre>zappashell
zappashell> python -m venv ve
zappashell> source ve/bin/activate 
(ve) zappa> pip install -r requirements.txt</pre> 

As mentioned above you can use Docker, to do so 

<pre>docker built -t prowl</pre>

Well, looks like you're all set, you're connected to Lambda, and have Zappa going! Congratulations for setting up an aspect of automation, but sometimes, it isn't this seamless, we've ran into this problem -- specifically myself so in the next section I will be talking about the problem once you try and deploy using Django/Zappa. 

## Deploying errors | Fixes 

Okay, so we've setup our Django/Lambda enviornment, it looks a little something like this

<pre>{
    "dev": {
        "django_settings": "montana.settings",
        "s3_bucket": "prowl-test-code"
    }
}

Does this look okay? (default 'y') [y/n]: y

Done! Now you can deploy your Zappa application by executing:

    $ zappa deploy dev

After that, you can update your application code with:

    $ zappa update dev

To learn more, check out our project page on GitHub here: https://github.com/Miserlou/Zappa
and stop by our Slack channel here: http://bit.do/zappa

Enjoy!,
 ~ Team Zappa!</pre>
 
 As you see we are pointing Lambda to the proper S3 bucket, and everything seems dandy, so let's deploy the dev version
 
 <pre>zappa deploy dev</pre>
 
 Unfortunately we encounter an error 
 
 <pre>zappa deploy dev
Calling deploy for environment dev..
Warning! AWS Lambda may not be available in this AWS Region!
Warning! AWS API Gateway may not be available in this AWS Region!
Oh no! An error occurred! :(

==============

Traceback (most recent call last):
    [boring callback removed]
NoRegionError: You must specify a region.

==============

Need help? Found a bug? Let us know! :D
File bug reports on GitHub here: https://github.com/Miserlou/Zappa
And join our Slack channel here: https://slack.zappa.io
Love!,
 ~ Team Zappa!</pre>
 
 In this case, we have an umbrella of options to try and fix this, I'll go over what has worked the best for myself personally. So what you'll need to do is specify a default region using environment variables, the drawback of using Lambda in my opinion is you must do this for every console, so a little frustrating. 
 
 <pre>export AWS_DEFAULT_REGION=us-west-1</pre>
 
 You need to add the default region in your `~/.awd/credentials` file
 
 <pre>[default]
aws_access_key_id = prowl_access_key
aws_secret_access_key = prowl_access_key
region=us-west-1</pre>

Remember to becareful with the JSON, and make sure your commas are in the right place. Now let's try and deploy again 

<pre>zappa deploy dev</pre> 

This should deploy, but if it doesn't it might be Django's security features blocking it from actually deploying so try to open the `settings.py` file and change `ALLOWED_HOSTS` to

<pre>ALLOWED_HOSTS = [ '127.0.0.1', 'x6kb437rh.execute-api.us-west-1.amazonaws.com', ]</pre>

Then redeploy. 

## Django modules 

There's a couple of Django modules you should have already on your Mac, or PC. Assuming you're running Linux, you need to run 
<pre>pip install django-storages boto</pre> 

Going back to the `settings.py` file, you want to add the following lines

<pre>INSTALLED_APPS = (
          ...,
          'storages',
     )</pre> 
     
## Configuring the RDS security group 

By default newly created RDS Security Groups have no inbound access. So you need to make sure your RDS Security group has open TCP connections from your subnets associated with your AWS Lambdas.

| Type     | Protocol         | Port Range |
| ------------- |:-------------:| -----:|
| All TCP    | TCP | 5432 |

## Using built in Zappa commands to control Route53 

In Zappa there are some custom commands you can run which make it SUPER easy to navigate around a somewhat complex set of circumstances, I've made a table below to explain 

| DNS Provider   | CA         | Notes |
| ------------- |:-------------:| -----:|
| Route53    | AWS Certificate Manager | All AWS combo makes this ridiculous easy	|
| Route53    | Let's Encrypt           | Another good option that Zappa has smoothed the way |
| Other DNS  | Other                   | You've got some work to do |

You can edit the Zappa settings file, which is to configure all these gateways properly 

<pre>     "certificate_arn": "arn:aws:acm:us-west-1:738356466015:certificate/1d066282-ce94-4ad7-a802-2ff87d32b104",
        "domain": "www.getprowl.com",</pre>
        
Then

<pre>"dev": {
        "django_settings": "montana.settings", 
        "s3_bucket": "haus-coffee",
        "aws_region": "us-west-1",</pre>
        
Now obviously the bucket can be anything, in this case it's Haus Coffee based out of San Francisco. Now certify that Route53 passes all the tests that Zappa can perform, and hopefully you'll get it certified in your virtual environment

<pre>(ve) $ zappa certify dev
Calling certify for environment dev..
Are you sure you want to certify? [y/n] y
Certifying domain www.getprowl.com..
Created a new domain name with supplied certificate. Please note that it can take up to 40 minutes for this domain to be created and propagated through AWS, but it requires no further work on your part.
Certificate updated!
</pre>

## Lambda environment variables 

Your code can pull Prowl information/data dumps from the execution environment by using the built-in AWS Lambda environment variables, but there are also custom ways of doing this in Zappa. For example, you could add `SOME_LAMBDA_KEY` in the AWS console and retrieve it in your code, via 

<pre>import os
some_lamda_key = os.environ.get('SOME_LAMBDA_KEY')
# or get system values
aws_lambda_function_name = os.environ.get('AWS_LAMBDA_FUNCTION_NAME')</pre>

## Conclusion 

Hopefully after reading this, you have a general understanding of how Prowl is applying AWS to achieve our goal. Some of the things I covered above could have been done on a CDN like CloudFront or StackPath, but for the sake of time I won't be covering that here, maybe in a different documentation piece, just to cover all grounds for our developers depending on the path they want to take when using a VM/venv. This documentation is made generally for Prowl employees in a private repo (not the one in public, the one in private is filled with documentation) but this particular piece could potentially be applied to other situations, and for that reason I made this open to help others. 

Written by Montana Mendy (c) 2017 
https://www.getprowl.com
