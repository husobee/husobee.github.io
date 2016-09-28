---
layout: post
title: "Hacking Client Certs into httpstat"
date: 2016-09-28 05:00:00
categories: httpstat client-certificates
---

Today I was playing around with [httpstat][httpstat] which is described as
"like curl -v, with colours."  Very handy tool tells you timing information, 
headers and can dump the response body into an output file.  Super handy if you
are troubleshooting.

I decided to try [to hack in client credentials][httpstat-fork] which was pretty
fun.  As you can see I am really just the following additions:

{% highlight go %}
var (
	clientCert tls.Certificate
	caCert []byte
)

if clientCertFile != "" && clientKeyFile != "" {
	clientCert, err = tls.LoadX509KeyPair(clientCertFile, clientKeyFile)
	if err != nil {
		log.Fatalf("unable to load client cert and key pair: %v", err)
	}
	tr.TLSClientConfig.Certificates = []tls.Certificate{clientCert}
}

if clientCAFile != "" {
	caCert, err = ioutil.ReadFile(clientCAFile)
	if err != nil {
		log.Fatalf("failed to read client ca file: %v", err)
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(caCert)
	tr.TLSClientConfig.RootCAs = certPool
}
{% endhighlight %}

I am just loading the x509 key pair from the client cert file and key, and
adding the tls certificate to the list of the transport's tls config, as well
as add the client's ca signing certificate to the transport's tls config root 
ca.

[Issue is here][cc-issue] and hoping this is useful for anyone.

[httpstat]: https://github.com/davecheney/httpstat
[cc-issue]: https://github.com/davecheney/httpstat/issues/94
[httpstat-fork]: https://github.com/husobee/httpstat/
