
## 前言

核心一点就是申请的SSL证书，不能下载，然后在AWS上创建一个load balancer把ALB上的https证书配置指向申请的SSL证书配置，然后把LB上的https的443端口转发到AWS EC2的服务实例暴露的端口上，LB上的80端口也重定向到LB上的443端口。这样LB到EC2就是内网，公网到LB就是https。

## 简述

Step 1: Request and Validate an SSL/TLS Certificate

1.  Go to the Amazon Certificate Manager (ACM) in the AWS Management Console.
2.  Click "Request a certificate" and enter your domain name.
3.  Select the validation method you prefer, such as email validation or DNS validation.
4.  Configure your settings and review your request before submitting it.
5.  Repeat the process if you want to include any subdomains.

Step 2: Approve the Certificate

1.  Check your email for a message from ACM with a validation link.
2.  Click the link and approve the request.
3.  Wait for a confirmation email stating the certificate has been issued.

Step 3: Install the SSL/TLS Certificate on Your Load Balancer

1.  Go to the Amazon EC2 console and select your load balancer.
2.  Click "Listeners" and then "Add listener."
3.  Enter the port you want to use for SSL/TLS traffic (usually 443) and select the ACM certificate to use.
4.  Save your changes.

Step 4: Configure Your Load Balancer to Forward Traffic to Your EC2 Instance(s)

1.  In the EC2 console, select the target group for your instances.
2.  Click "Edit," configure the target group, and remember to specify the port your instances use.
3.  Return to your load balancer and click "Target groups" and then "Edit."
4.  Add your target group and save your changes.

Step 5: Verify that Your Certificate is Working Correctly

1.  Visit your website using https:// and make sure it displays correctly.
2.  Use an SSL checker tool to ensure that your certificate is valid and installed correctly.

## 正文


[Adding an SSL Certificate to an Application Load Balancer in AWS | by Troy Ingram | AWS in Plain English](https://aws.plainenglish.io/adding-a-ssl-certificate-to-an-application-load-balancer-in-aws-966f8b5033e)


