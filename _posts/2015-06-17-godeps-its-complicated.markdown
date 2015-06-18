---
layout: post
title: "Godeps... Its Complicated."
date: 2015-06-17 20:00:00
categories: golang dependencies
---

My status with godeps: "It's Complicated."

For some background, [godep is a go dependency management tool][godep].  Basically, at a 
very high level, godep culls all of the source code needed by your code out of 
your development environment, and into a virtual work-space within your code base.
Why would you EVER want to keep all of your dependency code within your source 
tree??  Well, there are plenty of reasons.

1. Reproducible builds - This can't be understated.  With all of your source 
code together, in one place, you can make the same exact product build regardless
of build environment, dev workstation, etc.
2. Version Pinning - YOU have control over what gets built into your application,
not your upstream dependency developer.  `go get` literally gets the latest of 
the dependency when it is run for the first time, so you have no idea what is 
going into your product.
3. Simplify builds - Your build system will thank you, as it doesn't have to go 
get every dependency, every time you perform a build.
4. Easier Auditing - You can make sure your dependency is what you expect it is.

All of these reason scream for the need for robust dependency management.  Enter
my fr-enemy godep.

I <3 godep.  Dependency management in go proper doesn't exist.  Godeps fills 
that void by allowing for commit level version pinning.  See Below:

{% highlight sh %}
[husobee@localhost dockerspew]$ godep save
[husobee@localhost dockerspew]$ ls Godeps
Godeps.json  Readme  _workspace
[husobee@localhost dockerspew]$ cat Godeps/Godeps.json
{
	"ImportPath": "github.com/husobee/dockerspew",
	"GoVersion": "go1.4",
	"Deps": [
		{
			"ImportPath": "github.com/BurntSushi/toml",
			"Comment": "v0.1.0-18-g443a628",
			"Rev": "443a628bc233f634a75bcbdd71fe5350789f1afa"
		},
		{
			"ImportPath": "github.com/Sirupsen/logrus",
			"Comment": "v0.7.3",
			"Rev": "55eb11d21d2a31a3cc93838241d04800f52e823d"
		},
...

{% endhighlight %}

Sweet!  I have every revision of every package my project needs to build successfully,
moreover I have the commit comment of said revision. This is very helpful on it's own. 
But wait, there is more:

{% highlight sh %}
[husobee@localhost Godeps]$ du _workspace
_workspace/src/gopkg.in/unrolled/render.v1/fixtures/amber/layouts
_workspace/src/gopkg.in/unrolled/render.v1/fixtures/amber
_workspace/src/gopkg.in/unrolled/render.v1/fixtures/basic/admin
_workspace/src/gopkg.in/unrolled/render.v1/fixtures/basic
_workspace/src/gopkg.in/unrolled/render.v1/fixtures/custom_funcs
_workspace/src/gopkg.in/unrolled/render.v1/fixtures
_workspace/src/gopkg.in/unrolled/render.v1
_workspace/src/gopkg.in/unrolled
_workspace/src/gopkg.in/yaml.v2
_workspace/src/gopkg.in
_workspace/src/golang.org/x/crypto/openpgp/packet
_workspace/src/golang.org/x/crypto/openpgp/errors
_workspace/src/golang.org/x/crypto/openpgp/s2k
...

{% endhighlight %}

Godep saved ALL of the required source to easily rebuild what I have at this exact moment
in space time.  WOW.  This is excellent.  No more need to cull all of the source code during
a build of the software anymore.  I can actually audit all of this code, and have and hold
these dependency artifacts for the rest of my days, and they will never break due to "what's changed"
again!

I >:@ godep.  Updating godeps is a huge pita.  Dependencies have dependencies, which have dependencies.
This is a fact of life.  Godep is smart enough to figure this out on a godep save, and godep update, but
it relies heavily on go get to do the heavy lifting when doing an update... If only all your dependencies
also used a dependency management system...  Here are the steps you have to follow when you want to make
an update to an existing package with godeps:

1. go get -u <package you want to update>
2. godep update <package you want to update>
3. rinse, repeat, for every package you want to update.

Not kidding.  If the dependencies of those packages you want to update have changed, or are no longer valid 
in your dev environment, you will have to go get -u them too, and update godeps with the above.  Ouch.

See, Godep uses your machine's $GOPATH to find all of the dependencies for your project.  They assume, probably
correctly, that the developer is always right and does his/her due diligence to make sure all of the packages
on one's gopath are correct, and working together in harmony.  bwahahaha.

Some developers I know, (yes I am talking about you), would rather throw caution to the wind, and blow away
the entire Godeps directory in the codebase, and perform a godep save to quickly deal with updates to dependencies.
Since the dependencies are stored in your codebase, this makes for some VERY interesting commits, for sure, and much 
squabbling and frustration for devs. >;)

Hope this was helpful to anyone.

[godep]: https://github.com/tools/godep
