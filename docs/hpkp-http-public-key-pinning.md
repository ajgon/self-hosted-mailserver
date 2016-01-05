HPKP: HTTP Public Key Pinning
=============================

HTTP Public Key Pinning, or HPKP, is a security policy delivered via a HTTP response header much like [HSTS](https://scotthelme.co.uk/hsts-the-missing-link-in-tls/) and [CSP](https://scotthelme.co.uk/content-security-policy-an-introduction/). It allows a host to provide information to a user agent about which cryptographic identities it should accept from the host in the future. This can protect a host website from a security compromise at a Certificate Authority where rogue certificates may be issued for your hostname.

What Is HPKP?
-------------

HPKP is a HTTP response header that allows a host to deliver a set of fingerprints to cryptographic identities that the User Agent (UA) should accept for the site going forwards. Much like HSTS, HPKP is a Trust On First Use (TOFU) mechanism and the UA must establish at least one secure session with the host to receive the header intact. Once the header has been received, the UA will cache and apply the policy as per the directives included in the header.

Why Do We Need HPKP?
--------------------

Any one of the Certificate Authorities (CAs) in your trust store can issue a certificate for your hostname and the browser will trust it implicitly. This is great in the sense that you can obtain a certificate from any CA of your choosing, but not so great when one gets compromised.

#### DigiNotar

Back in 2011 a Dutch CA called DigiNotar was hacked. After gaining access to their systems, the hackers managed to make their way to the CA servers and issued over 500 rogue certificates to themselves. Included in those certificates was one for the google.com domain. This certificate was used to launch a Man in The Middle (MiTM) attack against 300,000 Iranian users of Google services, including GMail. From the end user perspective, the attack was undetectable. The browser received a certifcate for the domain, it was valid and the chain of trust was intact. They had all the indicators of a secure connection but it was totally compromised. There was a lot of speculation about who was responsible for the hack, but from the host's perspective, their users were compromised without any way for them to have known. HPKP changes that.

#### StartCom

In 2008 and again in 2011 the StartCom CA was breached. The 2008 breach, carried out by a security researcher, resulted in rogue certificates being issued for the paypal.com and verisign.com domains. The 2011 breach, carried out by an unkown attacker, came very close to the crown jewel, the StartCom root key. With access to the StartCom root key, the attacker could have generated a certificate for any domain they like, but not only that, it would have led to the necessary revocation and reissue of every single certificate issued by StartCom. No small task!

What Does HPKP Do?
------------------

HPKP can be enforced with a very simple HTTP response header. By specifying the fingerprint of certain cryptographic identities, you can force the UA to only accept those identities going forwards. The most ideal solution is to include the fingerprint of your current TLS certificate and at least one backup. The backup can be the fingerprint of a Certificate Signing Request so that you don't have to purchase a backup certificate. If the private key of your certificate were ever compromised, you could use the CSR to request the signing of a new public key. For this to work, the CSR has to be created with a brand new RSA key pair and stored securely offline. As the fingerprint of the CSR was already in the HPKP header, you can switch out to the new certificate without a problem. Using this method, if any CA was ever compromised, even your own CA, any rogue certificates that were issued for your domain would not be accepted by a browser that had received the HPKP header. Because the fingerprint of the rogue certificate has not been received and cached by the browser, it will be rejected and a connection to the site won't be allowed.

How Do We Setup HPKP?
---------------------

Setting up and deploying HPKP is incredibly easy, you just have to make sure that you have appropriate backups in place. If you lose all of the backups then you only have until your current certificate expires to get a new policy out to all of your visitors!

#### Add Your Existing Certificate

The first step to creating a HPKP policy is to get the fingerprint of your current certificate.

`openssl x509 -pubkey < tls.crt | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64`

That looks like a fairly complicated command but it breaks down to be quite simple.

`openssl x509` We're using the OpenSSL x509 certificate utility which can perform a variety of tasks. All we will be using it for is to output some information about our certificate.
 `-pubkey` Output the Subject Public Key Info (SPKI) block in PEM format.
 `< tls.crt` The TLS certificate you want to output the information of.

This information is then piped into a new command with the `|` operator.

`openssl pkey` The OpenSSL pkey command allows keys to be converted between forms.
 `-pubin` Flag that we are providing a public key, as a private key is the default.
 `-outform der` Set the output format to DER.

This information is then piped again into the penultimate command.

`openssl dgst` The OpenSSL dgst command is used to output the digest of the provided file.
 `-sha256` Use the SHA256 hash on the input.
 `-binary` Output the signature in binary format.

Lastly, we want to pipe the signature into the `base64` command to get the fingerprint.

You will end up with a Base64 string that looks something like this:

`X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg=`

Make a note of your fingerprint as you will need it to construct the HPKP header later on.

#### Creating A Backup CSR

Because we're going to be telling the browser to only accept the specific identities we provide in the header, it's a good idea to have some backups. In fact, I'd say it was absolutely essential. Fortunately, you don't need to have backup certificates. Instead, you create some backup CSRs and include their fingerprints in the header. The only time you will need these is if your private key is compromsied and you need a new certificate, or at your next renewal. It's always a good idea to generate a new key when you renew but you *must* create the CSRs based on a new key. If your private key is compromised and the CSR was based on your current key pair, it's useless. I'm going to be creating 2 new keys and 2 CSRs so I have a robust backup plan. To start with, generate a new private key.

`openssl genrsa -out scotthelme.first.key 4096`

This command uses the `openssl genrsa` command to create a new RSA private key. The `-out scotthelme.first.key` specifies where we would like to save the key and `4096` sets how large the key should be in bits. Now we have a new private key, we need to generate a CSR for it.

`openssl req -new -key scotthelme.first.key -sha256 -out scotthelme.first.csr`

Using the `openssl req` (request) command we want to create a `-new` CSR using the `-key scotthelme.first.key`. The certificate needs to use the `-sha256` message digest and finally we specify where to save the CSR `-out scotthelme.first.csr`. There is then some information you need to provide for the CSR.

    Country Name (2 letter code) [AU]:UK
    State or Province Name (full name) [Some-State]:Lancashire
    Locality Name (eg, city) []:Clitheroe
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Scott Helme
    Organizational Unit Name (eg, section) []:IT
    Common Name (e.g. server FQDN or YOUR name) []:scotthelme.co.uk
    Email Address []:scotthelme@hotmail.com

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:

Change the information in the above example to suit your own and note that the last 2 fields can be left empty. Now that the CSR is generated, all we need is the fingerprint to include in the HPKP header and we're good to go.

`openssl req -pubkey < scotthelme.first.csr | openssl pkey -pubin -outform der | openssl dgst -sha256 -binary | base64`

Similar to the previous step on obtaining the fingerprint for the certificate, there are just a couple of differences.

`openssl req` Using the OpenSSL request command again.
 `-pubkey < scotthelme.first.csr` We want the public key from the CSR we just created.

That is then piped into another command to convert the key.

`openssl pkey` Using OpenSSL pkey again to convert between formats.
 `-pubin` Flag that we are providing a public key, as a private key is the default.
 `-outform der` Set the oputput format to DER.

The penultimate command to get the SHA256 digest.

`openssl dgst` The OpenSSL dgst command to hash the provided input.
 `-sha256` Hashed using SHA256.
 `-binary` Output in binary.

Finally we pipe that into the `base64` command to get the fingerprint.

`MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec=`

Now, you may have noticed that I used a naming convention in the files through that example that included `first`, because you guessed it, it's time to rinse and repeat for a second backup! Once you've run all the commands again with `second`, you should have another private key, CSR and fingerprint.

`isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg=`

#### Configuring NginX

The easiest part of configuring HPKP is adding the header to NginX. Open up the config file for your site and in the server block, add the following with substitutions for your own fingerprints.

    add_header Public-Key-Pins 'pin-sha256="X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg="; \
    pin-sha256="MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec="; \
    pin-sha256="isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg="; \
    max-age=10';

This adds a new HTTP response header in NginX, defines the 3 fingerprints that we have created above and finally sets a maximum age for the policy. Save the changes to your config and reload the NginX configuration.

`sudo service nginx reload`

I've used a very short `max-age` value of 10 seconds for testing purposes so that if something does go wrong, you can remove the header and the policy will expire very quickly, allowing you access to your site again. Once you're happy with the setup and that everything is working you can increase the `max-age` value to something more suitable like 6-12 months. But remember, the value is in seconds!

#### The Report Only Header

(**added 27th Jan**)

Thanks to Jerome ([@exploresecurity](https://twitter.com/exploresecurity/status/560020530800377857)) and his follow-up [article](http://www.exploresecurity.com/five-considerations-for-http-public-key-pinning-hpkp/), I'm adding in this section on a better way to test your HPKP header. Using the HPKP Report Only header (`Public-Key-Pins-Report-Only`), you can issue your HPKP policy and test the impact without the risk of a failed connection if you get it wrong. In the same way as the report only header for CSP works, the browser will receive the header and output any information about violations to the console and to the `report-uri`, if one is provided, but it will not block the connection. I have covered the `report-uri` directive further on in this blog if you want to implement it, otherwise your header would look something like this.

    add_header Public-Key-Pins-Report-Only
    'pin-sha256="X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg="; \
    pin-sha256="MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec="; \
    pin-sha256="isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg="; \
    includeSubdomains';

There are quite a few things to note with using the report only header, and indeed a few other elements of using HPKP in general, so I'd recommend Jerome's article linked above. Even with the testing that you can do with the report only header, I'd still recommend a short `max-age` value when deploying a policy into a live environment just as an added precaution. (`max-age` is missing from the example above as report only policies are not cached so the directive is ignored).

#### Including Subdomains In HPKP

There are 2 ways of dealing with subdomains that also utilise TLS on your site. You can have each domain issue its own unique HPKP policy that specifies the fingerprints for identities to be used on that domain, or, you can issue a HPKP policy at the top that will cascade down all subdomains by using the `includeSubdomains` directive. Each method has advantages and drawbacks.

    add_header Public-Key-Pins 'pin-sha256="X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg="; \
    pin-sha256="MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec="; \
    pin-sha256="isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg="; \
    max-age=10; includeSubdomains';

For example, if you were to navigate directly to test.scotthelme.co.uk, but scotthelme.co.uk was issuing the HPKP policy for that domain and all subdomains, you wouldn't receive the policy. The HPKP header from scotthelme.co.uk would contain the fingerprints for all certificates used on the site and all subdomains, so would act like a master policy. This policy would need to be issued across all subdomains to be effective. You'd also have to have a lot of backups in there to cover revocations if you were compromised and renewals when they come around. I'd say 2 backups per domain is a good idea. It'd be a cumbersome policy but a 'one size fits all' to be issued across all subdomains.

Issuing a specific HPKP header per subdomain results in a smaller header, but a little more management. You need to track the fingerprints for each subdomains certificate and backups and ensure that they are presented in the correct header. Once setup in this manner, none of the policies can contain the `includeSubdomains` directive or there is the potential to break access to subdomains. You would have a much nicer header containing only 3 fingerprints per subdomain and the management isn't so bad once setup.

Reporting Pin Validation Failures
---------------------------------

Now that we have HPKP setup, we can rest assured that our visitors won't fall victim to a MiTM attack where a rogue certificate is used. That's fantastic news, but wouldn't it be even better if we could not only prevent the attack, but know about it in real time? Well, you can! HPKP has a `report-uri` directive where you can specify a URI for the UA to POST a JSON formatted failure report to. If the UA tries to connect to your site and the certificate fails to meet the criteria of the HPKP policy, we want to know about it. The report format is specified in the [IETF Draft](https://tools.ietf.org/html/draft-ietf-websec-key-pinning-21#section-3).

    {
         "date-time": date-time,
         "hostname": hostname,
         "port": port,
         "effective-expiration-date": expiration-date,
         "include-subdomains": include-subdomains,
         "noted-hostname": noted-hostname,
         "served-certificate-chain": [
           pem1, ... pemN
         ],
         "validated-certificate-chain": [
           pem1, ... pemN
         ],
         "known-pins": [
           known-pin1, ... known-pinN
         ]
    }

You need to include the directive in the policy and provide a suitable URI that is capable of receiving and processing such reports.

    add_header Public-Key-Pins "pin-sha256='X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg='; \
    pin-sha256='MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec='; \
    pin-sha256='isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg='; \
    max-age=10; report-uri='https://report.scotthelme.co.uk'";

Now, it goes without saying that the authenticity of these reports can never be assured. We have no way to prevent forged reports being delivered and reports may even contain malicious content like SQL Injection or XSS attempts. They should be treated with care and investigated with that in mind. There's also another problem. Well, actually, there could be a few. First, if an attacker has enough access to be able to MiTM you with a rogue certificate, they're going to be able to kill access to the URI used for reporting HPKP validation failures. Second, I use HSTS on my main domain and I'm even [HSTS preloaded](https://scotthelme.co.uk/hsts-preloading/) into Chrome, Firefox and Safari. As a result, HSTS is enforced on all my subdomains with the `includeSubdomains` directive in my HSTS policy. That also means that we should really be issuing a HPKP Policy on all subdomains if we're doing it right. Well, if we're reporting HPKP validation failures, how would we communicate with the report URI?

> Hosts may set report-uris that use HTTP or HTTPS. If the scheme in the report-uri is one that uses TLS (e.g. HTTPS), UAs MUST perform Pinning Validation when the host in the report-uri is a Known Pinned Host; similarly, UAs MUST apply HSTS if the host in the report-uri is a Known HSTS Host.

Of course, the attacker may only have a certificate for our main domain and the reports may be sent just fine, or the attacker could just block access to the URI, but it presents an interesting predicament. The draft does go on to say:

> In any case of report failure, the UA MAY attempt to re-send the report later.

Perhaps for now this is really the best option the UA has available. We are preventing the attack from taking place with the HPKP header, it would just be nice to know about it so that steps could be taken to resolve the issue.

Conclusion
----------

It's easy to make your site inaccessible if you get HPKP wrong, but, the same applies to any configuration change really. That said, it's also incredibly easy to do right and hugely improve your security stance. The 3 fingerprints we have set in the HPKP header are the *only* certificates that a browser will now accept for your site. As mentioned earlier, if a CA gets popped and issues rogue certificates for your domain, any browser that has received this header will reject the rogue certificates. This also means that you need to be cautious about how you backup the newly created keys and CSRs. If your server is compromised, it's no use having them on there as they will be compromised too. They need to be taken off the server and stored in a safe location for when they are needed. If your private key is compromised or your certificate is up for renewal, you will need to use one of the CSRs to obtain a new certificate. At that point, you switch to the new certificate, remove the fingerprint for the certificate that's on the way out, generate a new key pair and CSR, get the fingerprint and put that into the header so you always have 2 backup options. Anyone who has previously cached the policy will accept the new certificate as it was one of your backups and this is how you move forwards.

HPKP And Google Chrome
----------------------

Whilst writing this blog I came across a bug in how Google Chrome handles the HPKP header. You can find details on the bug [here](https://scotthel.me/r3j). The problem was around the handling of the `max-age` and `includeSubdomains` directives between HSTS and HPKP policies. If one policy had one of these values set, it would apply to the other. This resulted in my 10 second `max-age` value in the HPKP policy being ignored and the 31536000 value from my HSTS policy being applied across both. The same thing also happened for my subdomains. As it was now applying the fingerprints for scotthelme.co.uk to all my subdomains, they became inaccessible and the policy had an incredibly long duration too. For anyone who visited my site in the brief window that I had issued the policy, they would not have been able to access my subdomains until the patch for the issue made it to the stable build (or some time soon if they were running [Canary](https://www.google.com/chrome/browser/canary.html)), or I included the fingerprints for all of my subdomain certificates and their backups in the policy issued on the scotthelme.co.uk domain at the top. Not an ideal solution but HPKP policy is now being applied properly and here is my final policy.

    add_header Public-Key-Pins "pin-sha256='X3pGTSOuJeEVw989IJ/cEtXUEmy52zs1TZQrU06KUKg="; \
    pin-sha256="MHJYVThihUrJcxW6wcqyOISTXIsInsdj3xK8QrZbHec="; \
    pin-sha256="isi41AizREkLvvft0IRW4u3XMFR2Yg7bvrF7padyCJg="; \
    pin-sha256="aVNttuSMXJLW/fZOCaR/27b8k6ll5VDuoaVZf2rHIFU="; \
    pin-sha256="I/bAACUzdYEFNw2ZKRaypOyYvvOtqBzg21g9a5WVClg="; \
    pin-sha256="Y4/Gxyck5JLLnC/zWHtSHfNljuMbOJi6dRQuRJTgYdo="; \
    pin-sha256="9Eef0hLnaqy+EhIy8HnUCqFOKzTIWlNNZ5iCJlqT01E="; \
    pin-sha256="t3EPvqF+7XoKypCPHyN1b5uey7zTfIGDHn4oBWz2pds="; \
    pin-sha256="zqbcEslrpiH0bA9uhNyl2ovpLEfGJQM/QvZSVumMFJ8="; \
    includeSubdomains; max-age=31536000' always;
