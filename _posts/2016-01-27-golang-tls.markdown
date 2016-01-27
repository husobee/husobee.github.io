---
layout: post
title: "Golang HTTPS Hello Exchange Hook"
date: 2016-01-27 08:00:00
categories: golang tls
---

I was asked recently to log all client cipher suite capabilities as well as the User-Agent.  I mean, come on, 
how cool to be able to track TLS negotiation capabilities and facet on User-Agent.  This type of data would give
a lot of insight into how our customers connect to us with TLS.  In this blog I attempt to explain my thought 
process in coming up with a solution to this problem.

## What does the STDLIB afford?

The first stop to figuring this out for me was to look at the standard library [net/http][net-http] package for
clues on what data we could get for the HTTPS connection handshake.  Looking at the [Server][conn-state] there is
an interesting attribute called `ConnState` which is a function that takes a `net.Conn` implementation, as well
as a `http.ConnState` which can be NEW, ACTIVE, etc.  This function is called before the http.Server calls the 
request handlers by the `Server.Serve` method.

Armed with this I created a ConnState Hook function and passed it into the initialization of my server:

{% highlight go %}
func connStateHook(c net.Conn, state http.ConnState) {
	if state == http.StateActive {
		if cc, ok := c.(*tls.Conn); ok {
			state := cc.ConnectionState()
			switch state.Version {
			case tls.VersionSSL30:
				log.Println("negotiated to Version: VersionSSL30")
			case tls.VersionTLS10:
				log.Println("negotiated to Version: VersionTLS10")
			case tls.VersionTLS11:
				log.Println("negotiated to Version: VersionTLS11")
			case tls.VersionTLS12:
				log.Println("negotiated to Version: VersionTLS12")
			default:
				log.Println("negotiated to Unknown TLS version")
			}
		}
	}
}

func main() {
	s := &http.Server{
		Addr:      ":1234",
		ConnState: connStateHook,
		Handler:   nilHandler,
		TLSConfig: &tls.Config{
			GetCertificate: getCertificateHook,
		},
	}
	s.ListenAndServeTLS("cert.pem", "key.pem")
}
{% endhighlight %}

By type casting the parameter `net.Conn` to a `tls.Conn` type we are then able to get at the methods of the tls.Conn
such as `ConnectionState`.  ConnectionState provides access to [various state parameters][tls-conn-state] such as the 
negotiated TLS version the client and server have come to.  This is pretty interesting, but not exactly the data we want,
we actually want to see what the client is _capable_ of negotiating to...

Looking a bit deeper into the [crypto.tls][tls], specifically in [tls.Config][tls-config], I was able to find ANOTHER awesome
callback hook called `GetCertificate` which takes a `*ClientHelloInfo` and returns a `*Certificate` and error.  Looking at the
comments:

{% highlight go %}

        // GetCertificate returns a Certificate based on the given
        // ClientHelloInfo. It will only be called if the client supplies SNI
        // information or if Certificates is empty.
        //
        // If GetCertificate is nil or returns nil, then the certificate is
        // retrieved from NameToCertificate. If NameToCertificate is nil, the
        // first element of Certificates will be used.

{% endhighlight %}

This will work great for our purposes, and is a pretty neat callback.  Basically we would be able to get a custom certificate based on the 
Client's Hello handshake.  _BUT_ if we instead return a nil pointer for `*Certificate` the tls library will go about figuring out the correct
Certificate itself, which effectively provides us a free hook into the handshake ClientHelloInfo.

Perfect.  Updating our code:

{% highlight go %}
func getCertificateHook(helloInfo *tls.ClientHelloInfo) (*tls.Certificate, error) {
	o := &output{}
	for _, suite := range helloInfo.CipherSuites {
		if v, exists := CipherSuiteMap[suite]; exists {
			o.SupportedSuites = append(o.SupportedSuites, v)
		} else {
			o.SupportedSuites = append(o.SupportedSuites, fmt.Sprintf("Unknown, 0x%x", suite))
		}
	}

	for _, curve := range helloInfo.SupportedCurves {
		if v, exists := CurveMap[curve]; exists {
			o.SupportedCurves = append(o.SupportedCurves, v)
		} else {
			o.SupportedCurves = append(o.SupportedCurves, fmt.Sprintf("Unknown, 0x%x", curve))
		}
		// http://www.iana.org/assignments/tls-parameters/tls-parameters.xml#tls-parameters-8
	}
	for _, point := range helloInfo.SupportedPoints {
		// http://tools.ietf.org/html/rfc4492#section-5.1.2).
		o.SupportedPoints = append(o.SupportedPoints, fmt.Sprintf("0x%x", point))
	}

	j, _ := json.Marshal(o)
	log.Println(string(j))
	return nil, nil
}

{% endhighlight %}
 
In the above code we are ranging over the `CipherSuites` and adding to a list of supported Suites for our log message. While 
we have access, we are also grabbing the list of Supported Curves and Points for the TLS handshake. [The Gist is Here][gist]

As you can probably see, there isn't really a way of linking the ClientHelloInfo with the http.Request in the standard library.
As it isn't a "normal" thing to have access to.  I had to change the TLS stdlib to accomodate the request, which isnt great, and 
I am not happy with it, but it works for the time being.


[gist]: https://gist.github.com/husobee/6e9f998653d66f7481da
[net-http]: https://golang.org/pkg/net/http/
[conn-state]: https://golang.org/pkg/net/http/#Server
[tls-conn-state]: https://golang.org/pkg/crypto/tls/#ConnectionState
[tls]: https://golang.org/pkg/crypto/tls
[tls-config]: https://golang.org/pkg/crypto/tls/#Config
