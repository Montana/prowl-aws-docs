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

![AWS](http://www.getprowl.com/assets/images/flowchart.png)

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

Okay, now that you're connected to Lambda, and you have hopefully grabbed our custom `django-storages` in the private repo, make sure Zappa has ALL of it's dependencies installed before you start using it 

<pre>zappashell
zappashell> python -m venv ve
zappashell> source ve/bin/activate 
(ve) zappa> pip install -r requirements.txt</pre> 
