---
layout: post
title: "Secure Programming - Web Applications"
date: 2017-08-28 15:30:00
categories: school secure-programming
---

I know... It has been a while since I have posted anything, and for that I am
sorry.

I have been accepted into a Masters program in Computer Science concentration
in Information Security, and I am going to use this blog to document some of
my learning and experiences, which might hopefully help some.

## Secure Programming - Web Applications

There are quite a few web application security anti-patterns that one needs to
watch for during a peer review, or initial implementation.  Below are broken
into sections some big vulnerabilities to watch out for when developing a web
application.

### SQL Injection

Yes, we will start with the big one.  [XKCD][bobbie-tables] has a pretty iconic
comic on this topic.  So much so that I cannot hear the words SQL Injection
without pulling this comic into the forefront of my head.

SQL injections are an attack vector in which the attacker can gain unauthorized
access to a SQL database due to the application's failure to restrict the user
input.  Since SQL is a statement based language, which includes a statement
terminator for each statement, it is fairly easy for a malicious user to change
their user specified input in such a way to take advantage of this.

Say you have a web application that takes in a userID as a parameter, and returns
the name of the user with that userID.  One could imagine a web application
handler looking something like this:

{% highlight go %}
func GetUserNameHandler(w http.ResponseWriter, r *http.Request) {
	// get user id from query string
	userID := r.URL.Query().Get("userID")
	// perform SQL query to get name for each user where userID matches
	row, err := db.QueryRow("SELECT name FROM users WHERE userID=" + userID)
	if err != nil {
		// failure
		log.Error(err)
	}
	var name string
	// scan the name
	if err := rows.Scan(&name); err != nil {
		log.Error(err)
	}
	defer rows.Close()
	// write header
	w.WriteHeader(200)
	// write result
	w.Write(name)
}

{% endhighlight %}

When we call the handler with `curl localhost/?userID=2344'; DROP TABLES; --'` the
handler's SQL query will perform a string replacement inside the handler code 
which will cause our new SQL statement to the database to look more like 
`SELECT name FROM users WHERE userID='2344'; DROP TABLES; --` which is pretty
serious.

It is very VERY important to both validate, and sanitize user inputs.  Had we 
used the prepared query functionality within golang's SQL package, our SQL query
would have been sanitized, in that all SQL specific tokens such as `'` would have
been escaped for us.  We also should add a validation function that will check
the query string parameter to make sure it is a valid input.

### XSS and Response Splitting

XSS stands for Cross Site Scripting, which involves injecting malicious code
into a web page, of which the browser will execute when the page is rendered.
XSS has two different flavors, which in reality both accomplish the same thing.
There are Reflective and Persistent persuasions which will be defined below:

Reflective XSS is an attack vector in which a malicious user can formulate a
payload as the user input to a target web application, which will be immediately
presented to an unsuspecting user's browser for rendering and execution.  By
embedding a script into an unchecked user input a potential user effectively
re-writes the browser rendered code.  This is important because any element or
attribute which is available to the client script is also available to the injected
malicious script.  Such as Cookies, Session information, Local storage for that
domain, etc.  This information can easily be ex-filtrated from the browser and sent
to any other service.

Alternatively, Persistent XSS is pretty much the exact same attack, except the
malicious script is embedded into the application because the script was stored
within the servers persistence layer.  Example being user input for a comment
system on a blog, where the user is allowed to enter HTML as their comment, and
the comment is persisted within the database.

The end goal of the malicious user is to run their script within your browser,
effectively as your user, which is pretty important.  Below is an example of a
handler that could potentially allow an XSS attack:


{% highlight go %}
func GetUserNameHandler(w http.ResponseWriter, r *http.Request) {
	// get user id from query string
	userID := r.URL.Query().Get("userID")
	// perform SQL query to get name for each user where userID matches
	row, err := db.QueryRow("SELECT name FROM users WHERE userID=", userID)
	if err != nil {
		// failure
		log.Error(err)
	}
	var name string
	// scan the name
	if err := rows.Scan(&name); err != nil {
		log.Error(err)
	}
	defer rows.Close()
	// write header
	w.WriteHeader(200)
	// write result
	fmt.Fprintln(w, `<HTML><BODY>%s is named %s<BODY><HTML>`, userID, name)
}

{% endhighlight %}

Now all the malicious user has to do is have an unsuspecting user click on 
a link that looks like this: `service.whatever/?userID=<script>alert("HAXED")</script>`

You can see above how the script will just be reflected back a the unsuspecting
user who clicked on the link.

Response Splitting is another type of attack that is similar in nature.  Response
Splitting takes advantage of HTTP's plain text nature in that the attacker will try
to inject new headers in the HTTP response.  A typical HTTP response will look
like this:

{% highlight text %}
HTTP/1.1 301 Moved Permanently
Location: https://husobee.github.io/
Content-Type: text/html
Content-Length: 174
{% endhighlight %}

Now if a user knew that the Location response header actually came from an input
to the web server, such as `/?redirect_url=...` then a user could do all kinds of
interesting things.  Lets imagine the user made a link such as this one `service.whatever/?redirect_url=https://evilservice.whatever/%0D%0AXMore%2DData%3A%20haha`

Given we use the following code:

{% highlight go %}
func RedirectUser(w http.ResponseWriter, r *http.Request) {
	// get user id from query string
	location := r.URL.Query().Get("redirect_uri")
	// write header
	w.Header.Add("Location", location)
	w.WriteHeader(301)
	// write result
	fmt.Fprintln(w, "moved")
}

{% endhighlight %}

If we use the malicious link above against this handler, you will see the following
response headers come back!

{% highlight text %}
HTTP/1.1 301 Moved Permanently
Location: https://evilservice.whatever/
More-Data: haha
Content-Type: text/html
Content-Length: 174
{% endhighlight %}

Notice the new header we were able to inject, More-Data: haha.

The real moral of these XSS hacks are to point out that user data is dangerous.
You should not, and cannot accept user data without validation, and sanitation.
You will be asking for trouble unless you treat every single user input as
potentially dangerous and serious.

### CSRF

CSRF stands for Cross Site Request Forgery, and can potentially happen just by
visiting a site.  Given that you have a service endpoint that performs a delete
of your user account, say `GET service.whatever/my-account/delete`.  A malicious
user could create a website and link to `service.whatever/my-account/delete`, and
upon clicking, it will perform the call to delete your account.

The best way to protect against this sort of attack is to firstly use any other
HTTP verb, such as POST to perform altering actions to your service.  Moreover you
need to have some form of shared secret between the server and each request.  This
CSRF token will act as a check to make sure that you are submitting the request
from a page that is trusted by the server.

What I usually do is have a session cookie which will reference the CSRF token in
a data store the server can access.  Then on every form that is submitted, or XHR
request that is made part of the submitted data is an encrypted version of that
same CSRF token.  This way you will know that the request actually came from a page
you trust.

[bobbie-tables]: https://www.xkcd.com/327/
