
与APNs服务器建立连接，需要相关的证书，在苹果官网认证，最终的客户端SDK需要有一个p12的证书。

1. Create a new directory;
	mkdir Apple\ Enterprise
	cd Apple\ Enterprise

2. Generate a certificate signing request
	openssl req -nodes -newkey rsa:2048 -keyout aps.key -out CertificateSigningRequest.certSigningRequest

3. With the information like so (ensure you give it a password):
	Country Name (2 letter code) [AU]:GB
	State or Province Name (full name) [Some-State]:London
	Locality Name (eg, city) []:
	Organization Name (eg, company) [Internet Widgits Pty Ltd]:Total Onion Ltd
	Organizational Unit Name (eg, section) []:
	Common Name (e.g. server FQDN or YOUR name) []:Total Onion Enterprise
	Email Address []:

4. Login to developer.apple.com, go to:
	"Member Center" -> "Certificates, Identifiers & Profiles > Identifiers > Select App IDs, Scroll down select "Add/Edit" button Besides Push Notification

5. Go through the wizard, selecting the certificate type, and uploading the .csr.

6. Download the .cer file, saving it to the folder created in step 1

7. Convert the .cer file to a .pem file:
	openssl x509 -in aps.cer -inform DER -out aps.pem -outform PEM

8. Convert the .pem to a .p12:
	openssl pkcs12 -export -inkey aps.key -in aps.pem -out aps.p12

9. You can now use the aps certificate on OneSignal or any other similar platforms.

## 参考

[Making Apple Apple Push Notification certificates on Linux (github.com)](https://gist.github.com/ajithrn/f19b4aae6dafe79bb5bde6c10b0bca2f)

[Making Apple Developer certificates on Linux (github.com)](https://gist.github.com/boodle/77436b2d9facb8e938ad)

[Making an APNs Certificate with Password - Apptentive Customer Learning Center](https://learn.apptentive.com/knowledge-base/making-an-apns-certificate-with-password/)

[Generate APNS certificate for iOS Push Notifications | by Ankush Aggarwal | Zero Equals False | Medium](https://medium.com/zero-equals-false/generate-apns-certificate-for-ios-push-notifications-85e4a917d522)



