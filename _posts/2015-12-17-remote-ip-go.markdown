---
layout: post
title: "Remote IP Address with Go"
date: 2015-12-17 08:00:00
categories: golang ip-address
---

# We need remote ips

Yesterday I was tasked with figuring out how to properly ascertain remote web
client IP Addresses in our golang web services.  This sounds like a stupidly 
easy exercise, and golang MUST have a fancy utility for this right?  Nope. 
Turns out getting the remote IP Address is really hard.

# The naïve approach

Within the golang `net.Request` structure there is a `RemoteAddr` attribute, 
which unsurprisingly contains the remote address of the requester.  Job done,
right?  Well not really if you use any form of reverse proxy or load balancer 
for your application, which we do.  This will always appear to the go server
as if every request is coming from the load balancer, which would be terrible
if you wanted to use this as a throttling metric of any kind.  So, we can 
immediately throw this out as non-useful for our purposes, unless you have one
go server running not behind any form of reverse proxy.

That thrown out as we run our server behind nginx as a reverse proxy, we can
start looking at various headers that contain better information.  I came across
two headers that are potentially useful, `X-Real-Ip` and `X-Forwarded-For`.

`X-Forwarded-For` you can think of as a request hop log.  Every proxy the request
traverses, that proxy is supposed to append the origination IP address it sees
to this header.  For example, if you have a home router proxy (ip1) and the 
service has a load balancer (ip2) your application would have an `X-Forwarded-For`
value of `ip1, ip2`

Very naïvely, we can just pull the first record off of this header and 
that would be the client's ip address, right?

{% highlight go %}

func getIPAdress(r *http.Request) string {
    var ipAddress string
    for _, h := range []string{"X-Forwarded-For", "X-Real-Ip"} {
        for _, ip := range strings.Split(r.Header.Get(h), ",") {
            // header can contain spaces too, strip those out.
            realIP := net.ParseIP(strings.Replace(ip, " ", "", -1))
            ipAddress = ip
        }
    }
    return ipAddress
}

{% endhighlight %}

# A better solution

What if the client uses a client side proxy for web requests?  If that happens
the first entry in the `X-Forwarded-For` header could potentially be in a "private"
sub-net, not globally accessible.  Below is a better solution that filters out
private sub-nets, as well as multi-cast address space, and localhost address space:

{% highlight go %}

//ipRange - a structure that holds the start and end of a range of ip addresses
type ipRange struct {
    start net.IP
    end net.IP
}

// inRange - check to see if a given ip address is within a range given
func inRange(r ipRange, ipAddress net.IP) bool {
    // strcmp type byte comparison
    if bytes.Compare(ipAddress, r.start) >= 0 && bytes.Compare(ipAddress, r.end) < 0 {
        return true
    }
    return false
}

var privateRanges = []ipRange{
    ipRange{
        start: net.ParseIP("10.0.0.0"),
        end:   net.ParseIP("10.255.255.255"),
    },
    ipRange{
        start: net.ParseIP("100.64.0.0"),
        end:   net.ParseIP("100.127.255.255"),
    },
    ipRange{
        start: net.ParseIP("172.16.0.0"),
        end:   net.ParseIP("172.31.255.255"),
    },
    ipRange{
        start: net.ParseIP("192.0.0.0"),
        end:   net.ParseIP("192.0.0.255"),
    },
    ipRange{
        start: net.ParseIP("192.168.0.0"),
        end:   net.ParseIP("192.168.255.255"),
    },
    ipRange{
        start: net.ParseIP("198.18.0.0"),
        end:   net.ParseIP("198.19.255.255"),
    },
}


// isPrivateSubnet - check to see if this ip is in a private subnet
func isPrivateSubnet(ipAddress net.IP) bool {
    // my use case is only concerned with ipv4 atm
    if ipCheck := ipAddress.To4(); ipCheck != nil {
        // iterate over all our ranges
        for _, r := range privateRanges {
            // check if this ip is in a private range
            if inRange(r, ipAddress){
                return true
            }
        }
    }
    return false
}

func getIPAdress(r *http.Request) string {
    var ipAddress string
    for _, h := range []string{"X-Forwarded-For", "X-Real-Ip"} {
        for _, ip := range strings.Split(r.Header.Get(h), ",") {
            // header can contain spaces too, strip those out.
            ip = strings.TrimSpace(ip)
            realIP := net.ParseIP(ip)
            if !realIP.IsGlobalUnicast() || isPrivateSubnet(realIP) {
                // bad address, go to next
                continue
            } else {
                ipAddress = ip
                goto Done
            }
        }
    }
Done:
    return ipAddress
}

{% endhighlight %}

As you can see the above is a lot of infrastructure, but now, we are able to get
an ip address that is our first globally visible IP address the request originates
from.  Granted this isn't perfect, because a malicious user could clearly create
fake X-Forwarded-For header records to play chaos monkey, but I guess nothing in
life is perfect.


## UPDATE!

After hearing from a friend about this mechanism it is pretty clear the above
mechanism could be exploited by griefers, he had a suggestion.  Instead of walking
from the left to right, walk backwards through the number of ip addresses by the
number of proxies you have in your environment to the internet.  That way, you 
will be adverse to any mucking with the `X-Forwarded-For` header by the client. 
Below is the updated version of the Above implementing this:

{% highlight go %}

func getIPAdress(r *http.Request) string {
    for _, h := range []string{"X-Forwarded-For", "X-Real-Ip"} {
        addresses := strings.Split(r.Header.Get(h), ",")
        // march from right to left until we get a public address
        // that will be the address right before our proxy.
        for i := len(addresses) -1 ; i >= 0; i-- {
            ip := strings.TrimSpace(addresses[i])
            // header can contain spaces too, strip those out.
            realIP := net.ParseIP(ip)
            if !realIP.IsGlobalUnicast() || isPrivateSubnet(realIP) {
                // bad address, go to next
                continue
            }
            return ip
        }
    }
    return ""
}

{% endhighlight %}
