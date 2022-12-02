# Static Website Setup on Amazon Cloud using Cloudfront S3 Route53 ACM for SSL

In this post, you will learn how to:

1. Create an S3 bucket and set it up for static website hosting
2. Create a record set in Route 53
3. Set up a CloudFront distribution and link it with a custom domain
4. Secure the connection via SSL and AWS Certificate Manager

The diagram above is a graphical representation of the infrastructure we will be setting up. We will be hosting our static website content on Amazon S3 bucket. Then, we will setup a custom domain name with AWS Route53 as our DNS, create SSL certificate to secure our content and finally, configure Cloudfront CDN for content caching and distribution.
We are going to create the static website in two ways - 

1. Serve the website publicly via HTTP protocol using S3 website endpoint.
2. Serve the website publicly via HTTPS protocol using SSL from ACM and Cloudfront.

**Prerequisite:**

- A domain name (You can get a free domain from [Freenom](http://www.freenom.com/en/index.html) )

- An Amazon account (Register for a [AWS Free Tier Account](https://aws.amazon.com/free/?trk=712ee378-d73b-4293-9bad-8ce09671ea7c&sc_channel=ps&s_kwcid=AL!4422!3!444219541826!e!!g!!aws%20free%20tier&ef_id=Cj0KCQiAvqGcBhCJARIsAFQ5ke7C8AzjS-yGr_Qv0zWtmc47RXWusLo-c_1IQ2XUaQkm0WayVy38AY0aAqXBEALw_wcB:G:s&s_kwcid=AL!4422!3!444219541826!e!!g!!aws%20free%20tier&all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Types=*all&awsf.Free%20Tier%20Categories=*all) )

- A static website (To follow along, you can clone this from [my Github repo](https://github.com/oayanda/static-website) )

Let's get started!

## Static Website Setup with HTTP protocol using S3 website endpoint.
**Setup S3 bucket**
Login into your AWS account, select your region at top right of your dashboard and search for S3 in the top search bar.

![s3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t21wr03eryknevga6y88.png)
1. Click on create bucket.
We are going to create two buckets. (Note: This must be the name of your domain and or your subdomain, for example `yourdomainname.com` and `www.yourdomainname.com`)

2. Enter your bucket name. AWS S3 is a global resource hence, your bucket name must be globally unique.
3. Verify and select the region nearest to you or where you would like to serve your content from.

![name](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0dzjode82csa7cmb1nnq.png)
4. Uncheck the box under _Block Public Access settings for this bucket_ and click on the checkbox to acknowledge your content will be visible to the public

![s3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m8no7urlraai1csl7wzi.png)
5. To upload the website content, Click on the bucket and click upload. Navigate to the your files and folders of website from your computer and drag and drop. Then, click upload.

![upload](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6kt6s3cjl9ygr1obkibk.png)
6. To configure the bucket, click on the bucket and then _**properties**_ tab.

![properties](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xtc9z1m0aqeem1g8z20p.png)
7. On the _**properties**_ tab scroll down to the _Static website hosting_ section, click on edit and select _Enable_ , under _Index document_, enter **index.html** and click on save changes.

![web](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pz83mfbll7inib2u0mj9.png)
Finally, we are going to add permission for our content to be viewable to the public.
8. On the _permissions_ tab of the bucket, scroll down to _Bucket policy_ and click on _edit_

![policy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/munnembge0timz6xpl88.png)
Enter the code snippet below and click on save changes. Make sure to replace _**oayanda.com**_ with your _**domain name**_
```bash
{
  "Id": "Policy1669914787641",
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Stmt1669914786145",
      "Action": [
        "s3:GetObject"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::oayanda.com/*",
      "Principal": "*"
    }
  ]
}
```

![bucket](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yi19fdcxin7zusp9bwph.png)

Notice the red notification under your bucket name - _**Publicly accessible**_, now your website is publicly accessible.

![accessible](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qtsfi65j8ssp3ff051yi.png)

At this point, the public can view your website content via S3 object URL which look something like this `https://s3.eu-west-2.amazonaws.com/oayanda.com/index.html`.
You can see your S3 object URL by navigating to
`Amazon S3 > bucket name > index.html` 

![objecturl](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k1w6eriefegfr0xp7518.png)

This is fine but not memorable nor user friendly, hence we are going to setup route 53. Before we go further, let's create the second bucket `_**www.yourdomainname.com**_` for our subdomain. We create this because it's one of the two ways of accessing a domain. For example - `google.com` or `www.google.com`.

Don't worry this step is shorter than the first.
 
**Create second Bucket**
1. Navigate to Amazon S3 > Buckets > Create bucket
2. Enter your bucket name _**`www.yourdomainname.com`**_
3. Select and verify the region
4. Click on create bucket
5. Finally, navigate to the _properties_ tab of the bucket, scroll down to the _**Static website hosting**_ section, click on edit, select _**Enable**_ , under _**Hosting type**_, select **Redirect requests for an object**, under _**Host name**_ enter your domain name without the _**www**_ (_**`yourdomainname.com`**_), under _**Protocol**_ select _**HTTP**_ and click on save changes.

![second](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xc6vl9cfxci4xm4yy509.png)

The AWS S3 dashboard should look like this.
![bucket](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fmbrkaxa7dazligk1g0t.png)

**Configure S3 website Endpoint with Route 53**
On your AWS dashboard, search for Route 53 in the search bar.

![route53](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5atpjz91qioa944pnb27.png)
**Create a Hosted zone** 
1. Navigate to _Route 53_ > _Hosted zones_ > _Create hosted zone_
2. Under _**Domain name**_, enter your domain name (`yourdomainname.com`)
3. Ensure under _**Type**_, _**Public hosted zone**_ is selected and click _Create hosted zone_.
At this point, two records are created _**NS**_ (Name Servers) and _**CNAME**_(Canonical Name record).

![records](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kxdcscchigm653g1ltwd.png)

> _**Note:**_ _Replace or ensure that the four NS values are the same with your registered domain DNS._

![NS](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0pobhvtkbqxod4n7ufz0.png)

Finally, We will add two records, `_**yourdomain.com**_` and `_**www.yourdomian.com**_`

**Add Two A Records**
_For the first record_

1. Click on Create record
2. Leave _**Record name**_ field blank,
3. Toggle the Alias button
4. Under _**Route traffic to**_ select _**Alias to S3 website endpoint**_
5. Choose the region where your bucket was created
6. select the bucket

![bucket one](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/fhnzpc5xch13h3rw82tk.png)

_For the second record_
7. On the same page, click on _**Add another record**_
8. This time enter _**www**_ in the _**Record name**_ field
9. Toggle the Alias button
10. Under _**Route traffic to**_ select _**Alias to S3 website endpoint**_
11. Choose the region where your bucket was created and select the bucket.
12. Create records

It will take few seconds for the records to _SYNC_.

_**Test website in the browser**_
Enter your domain in the browser first try `yourdomain.com` and then `www.yourdomain.com`. Notice, when you entered `www.yourdomain.com` the browser redirects you to `yourdomain.com`. 
> Note: The web content is only in one bucket `yourdomain.com`, while the empty bucket `www.yourdomain.com` redirects to `yourdomain.com`. 

_If your website is showing, sometimes you might need to clear your browser's cache._                     

![website](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/i570lrdp6koqak0gmj5r.png)

Well, this is fine but not secure as you can see from the image above.

Let's make our website secure and increase the speed of distribution round the world by configuring a content delivery network - _**AWS Cloudfront**_.

## Static Website Setup with HTTPS protocol using AWS Cloudfront and AWS Certificate Manager ACM.

**Create two certificates using AWS certificate Manager (ACM)**

To make ensure your website is secure, you need a Secure Sockets Layer (SSL) certificate. On the search bar, type certificate manager and click enter.

![certificate](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m90ofbmgsefi9w4vbuig.png)

> Note: At the time of this post, you should create your certificates in _**North Virgin us-east-1**_.

Click on _**Request certificate**_, ensure _**Request a public certificate**_ is selected and click next

![Imag](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/v3e8pkws3hix3wg40g2r.png) 
3. Under _**Fully qualified domain name**_ enter your first domain name and click on _**Add another name to this certificate**_ to add the second domain name.
4. Ensure DNS validation is selected. (This is usually faster). This is done to ensure you are the owner of the domain name.
5. Click on Request.

![DNS](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/txno7yrtmi9ik9yqmpkv.png)
Click on _**View certificate**_ on the top of the page or click on _**List certificates**_

![Certs](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zzf8y8uwx8lo4i6rp2o2.png)
Click on _**Create records in Route 53**_ and click _**create records**_ at bottom of the page to confirm.

![records](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7l8nv595uol8yf9fcxso.png)

This will take some minutes but it's usually faster if purchased your domain from Amazon.

Verify certificate is issued.
![issued](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ddjtr4a07fap80vc1m4s.png)

**Create a bucket**

> Note With Cloudfront, the name of bucket must not necessarily be the same name as your domain name (for example - `oayanda.com`), it can be any globally acceptable name.

1. Enter a globally unique name - I will use oayanda-website and click _**create bucket.**_
2. Finally, upload the website content into the bucket.
We will come back here to give permission to cloudfront after setting it up. 

![upload](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q1lly1gtxxe8rztmzm1r.png)

**Setup AWS Cloudfront**

Navigate to AWS Cloudfront on your dashboard.
1. Click _**Create a CloudFront distribution**_
2. Under _**Origin domain**_ select the bucket that holds your website content
3. Under _**Origin access**_ , select _**Origin access control settings**_
4. Under _**Origin access control**_ , click on _**create control setting**_ this will allow amazon to automatically create an access policy for your S3 bucket for cloudfront.

![access](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/6oq6d9rtuxln1m3nbw52.png)
Accept the name and click _**create**_.

![create](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wdh8ujw94pi1kx2kg6cx.png)

Scroll down to _**Viewer**_ and select _**Redirect HTTP to HTTPS**_

![redirect](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zlxpa8gz48b0bsf274s4.png)

Scroll down to _**Alternate domain name (CNAME)**_, click on _**Add item**_ and enter two domain names `yourdomain.com` and `www.yourdomain.com`

![names](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8m2bal2lewo1socsyeou.png)
1. Scroll down to _**Custom SSL certificate**_ and select your certificate.
2. Scroll down to _**Default root object**_, enter default page name - in this _**index.html**_ and click on _**create distribution**_

![index](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/gsilqkgsyp60dwekn5ru.png)
Finally, we need to give permission to cloudfront on S3. 
1. Click on the newly created distribution and click on the _**origins tab**_
2. Under _**Origins**_ selected the origin and click on edit

![origin](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7bzgm8jn9mfjj80z8431.png)
Scroll down to _**Bucket policy**_ and click on _**Copy policy**_

![policy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/saoajecwwyulxvaflkgz.png)
Navigate to your S3 bucket _**permissions**_ tab, scroll down to _**Bucket policy**_, click on edit and paste the policy you copied from the cloudfront distribution and save changes.
![policy](/images/35.png)
Cloudfront will take some minutes to deploy the configuration

The last step is to add the records in Route 53.
_**Add Cloudfront Distribution to Route 53**_
1. Navigate to Route 53, click on create record
2. For the first record, leave the _**Record name**_blank
3. Toggle the Alias button and select _**Alias Cloudfront Distribution under**_ _**Route traffic to**_ field.
4. select the cloudfront distribution for `yourdomain.com`

![one](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r7a1xn139edjgdjyu353.png)
5. Click on _**Add another record**_
6. Enter _**www**_ in the _**Record name**_
7. Repeat step 3 to 4
8. Finally, click on _**create records**_.
The records will take some seconds to _**SYNC**_

_**Test your secured website with low-latency in the browser**_

Enter your domain in the browser first try `yourdomain.com` and then `www.yourdomain.com`.

![lock](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/60qnl4qd8fzu026ahrmr.png)
Notice the lock icon in the URL, this means your website is secure.

**Congratulation!!!** You have successfully deployed a static website via HTTP and HTTPS protocols in AWS.