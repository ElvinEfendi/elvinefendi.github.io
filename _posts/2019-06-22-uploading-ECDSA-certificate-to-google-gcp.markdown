---
layout: post
title: "uploading ECDSA TLS certificate to GCP"
date: 2019-06-22 13:00
comments: true
tags: gcp, google, tls, certificate, ecdsa
---

This is going to be a very short article, it's mostly for myself to remember how to upload ECDSA TLS certificate to GCP.

Few days ago I was experimenting with [Google's HTTPs Load Balancer](https://cloud.google.com/load-balancing/docs/https/). One of the
steps is to upload your TLS certificate to GCP so that Google's HTTPs Load Balancer (GCLB) can terminate TLS connections (it also has an opton
to use managed certificate, but that's not what I wanted to do). Similar to other GCP resources, this can be done using API, Web UI, or
`gcloud` command line tool. I was using `gcloud`.

[As noted](https://cloud.google.com/load-balancing/docs/ssl-certificates#gettingakeyandcertificate) GCP accepts only TSL certificates
using RSA-2048 or ECDSA P-256 encryption. The TLS certificate I wanted to upload was obtained from Cloudflare, to be installed in
my origin server. I confirmed that the private key satisfies GCP's requirements in terms of encryption algorithm by using

```
openssl ec -in key.pem -text -noout
```

Where `key.pem` was my private key. **When sharing keep in mind the output of this command includes sensitive information**.

After that I tried uploading the certificate:

```
gcloud compute ssl-certificates create elvin-test-cert --certificate cert.pem --private-key key.pem
```

However this did not work, and I got following error:

```
ERROR: (gcloud.compute.ssl-certificates.create) Could not fetch resource:
 - The SSL key could not be parsed.
```

I then tried to upload this in the web UI - but got the same error. After that I generated a self signed RSA-2048 certificate
and was able to successfully upload it.

Then I tried reformatting the private key I obtained from Cloudflare using:

```
openssl ec -in key.pem -out new_key.pem
```

And tried to upload this new key:

```
gcloud compute ssl-certificates create elvin-test-cert --certificate cert.pem --private-key new_key.pem
```

And success! It worked! So what was the issue with original private key? Because the fact that `openssl ec -in key.pem -text -noout` worked
on the original private key means the key itself is not corrupted. It turns out GCP requires ECDSA private key to strictly start with `-----BEGIN EC PRIVATE KEY-----` and end with `-----END EC PRIVATE KEY-----`. However the private key I got from Cloudflare did not have `EC` in those lines, reformatting
the key changed the comment from `-----BEGIN PRIVATE KEY-----` to `-----BEGIN EC PRIVATE KEY-----`.
