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

// readClientCert - helper function to read client certificate
// from pem formatted file
func readClientCert(filename string) []tls.Certificate {
	if filename == "" {
		return nil
	}
	var (
		pkeyPem []byte
		certPem []byte
	)

	// read client certificate file (must include client private key and certificate)
	certFileBytes, err := ioutil.ReadFile(clientCertFile)
	if err != nil {
		log.Fatalf("failed to read client certificate file: %v", err)
	}

	for {
		block, rest := pem.Decode(certFileBytes)
		if block == nil {
			break
		}
		certFileBytes = rest

		if strings.HasSuffix(block.Type, "PRIVATE KEY") {
			pkeyPem = pem.EncodeToMemory(block)
		}
		if strings.HasSuffix(block.Type, "CERTIFICATE") {
			certPem = pem.EncodeToMemory(block)
		}
	}

	cert, err := tls.X509KeyPair(certPem, pkeyPem)
	if err != nil {
		log.Fatalf("unable to load client cert and key pair: %v", err)
	}
	return []tls.Certificate{cert}
}

{% endhighlight %}

I am just loading the x509 key pair from the client cert file and key, and
adding the tls certificate to the list of the transport's tls config, as well
as add the client's ca signing certificate to the transport's tls config root 
ca.

Here is the call site of this helper function in the visit function:

{% highlight go %}
	ClientConfig = &tls.Config{
				ServerName:         host,
				InsecureSkipVerify: insecure,
				Certificates:       readClientCert(clientCertFile),
			}
{% endhighlight %}

[Issue is here][cc-issue] and hoping this is useful for anyone.
[PR is here][cc-pr] and hoping this is useful for anyone.

[httpstat]: https://github.com/davecheney/httpstat
[cc-issue]: https://github.com/davecheney/httpstat/issues/94
[cc-pr]: https://github.com/davecheney/httpstat/pull/99
[httpstat-fork]: https://github.com/husobee/httpstat/
