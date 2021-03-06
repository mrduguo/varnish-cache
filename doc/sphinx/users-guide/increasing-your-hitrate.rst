.. _users-guide-increasing_your_hitrate:

Achieving a high hitrate
------------------------

Now that Varnish is up and running, and you can access your web
application through Varnish. Unless your application is specifically
written to work behind a web accelerator you'll probably need to do
some changes to either the configuration or the application in order
to get a high hit rate in Varnish.

Varnish will not cache your data unless it's absolutely sure it is
safe to do so. So, for you to understand how Varnish decides if and
how to cache a page, I'll guide you through a couple of tools that you
will find useful.

Note that you need a tool to see what HTTP headers fly between you and
the web server. On the Varnish server, the easiest is to use
varnishlog and varnishtop but sometimes a client-side tool makes
sense. Here are the ones I use.

Tool: varnishtop
~~~~~~~~~~~~~~~~

You can use varnishtop to identify what URLs are hitting the backend
the most. ``varnishtop -i txurl`` is an essential command, showing you
the top txurl requests Varnish is sending towards the backend. You can
see some other examples of varnishtop usage in
:ref:`users-guide-statistics`.


Tool: varnishlog
~~~~~~~~~~~~~~~~

When you have identified the an URL which is frequently sent to the
backend you can use varnishlog to have a look at the request.
``varnishlog -c -m 'RxURL:^/foo/bar`` will show you the requests
coming from the client (-c) matching /foo/bar.

For more information on how varnishlog works please see
:ref:`users-guide-logging` or man :ref:`ref-varnishlog`.

For extended diagnostics headers, see
http://www.varnish-cache.org/trac/wiki/VCLExampleHitMissHeader


Tool: lwp-request
~~~~~~~~~~~~~~~~~

lwp-request is part of The World-Wide Web library for Perl. It's a
couple of really basic programs that can execute an HTTP request and
give you the result. I mostly use two programs, GET and HEAD.

vg.no was the first site to use Varnish and the people running Varnish
there are quite clueful. So it's interesting to look at their HTTP
Headers. Let's send a GET request for their home page::

  $ GET -H 'Host: www.vg.no' -Used http://vg.no/
  GET http://vg.no/
  Host: www.vg.no
  User-Agent: lwp-request/5.834 libwww-perl/5.834
  
  200 OK
  Cache-Control: must-revalidate
  Refresh: 600
  Title: VG Nett - Forsiden - VG Nett
  X-Age: 463
  X-Cache: HIT
  X-Rick-Would-Never: Let you down
  X-VG-Jobb: http://www.finn.no/finn/job/fulltime/result?keyword=vg+multimedia Merk:HeaderNinja
  X-VG-Korken: http://www.youtube.com/watch?v=Fcj8CnD5188
  X-VG-WebCache: joanie
  X-VG-WebServer: leon

OK. Let me explain what it does. GET usually sends off HTTP 0.9
requests, which lack the Host header. So I add a Host header with the
-H option. -U print request headers, -s prints response status, -e
prints response headers and -d discards the actual content. We don't
really care about the content, only the headers.

As you can see, VG adds quite a bit of information in their
headers. Some of the headers, like the X-Rick-Would-Never are specific
to vg.no and their somewhat odd sense of humour. Others, like the
X-VG-Webcache are for debugging purposes. 

So, to check whether a site sets cookies for a specific URL, just do::

  GET -Used http://example.com/ |grep ^Set-Cookie

Tool: Live HTTP Headers
~~~~~~~~~~~~~~~~~~~~~~~

There is also a plugin for Firefox. *Live HTTP Headers* can show you
what headers are being sent and recieved. Live HTTP Headers can be
found at https://addons.mozilla.org/en-US/firefox/addon/3829/ or by
googling "Live HTTP Headers".


The role of HTTP Headers
~~~~~~~~~~~~~~~~~~~~~~~~

Along with each HTTP request and response comes a bunch of headers
carrying metadata. Varnish will look at these headers to determine if
it is appropriate to cache the contents and how long Varnish can keep
the content.

Please note that when considering these headers Varnish actually
considers itself *part of* the actual webserver. The rationale being
that both are under your control. 

The term *surrogate origin cache* is not really well defined by the
IETF so RFC 2616 so the various ways Varnish works might differ from
your expectations.

Let's take a look at the important headers you should be aware of:

.. _users-guide-cookies:

Cookies
-------

Varnish will, in the default configuration, not cache a object coming
from the backend with a Set-Cookie header present. Also, if the client
sends a Cookie header, Varnish will bypass the cache and go directly to
the backend.

This can be overly conservative. A lot of sites use Google Analytics
(GA) to analyze their traffic. GA sets a cookie to track you. This
cookie is used by the client side javascript and is therefore of no
interest to the server. 

Cookies from the client
~~~~~~~~~~~~~~~~~~~~~~~

For a lot of web application it makes sense to completely disregard the
cookies unless you are accessing a special part of the web site. This
VCL snippet in vcl_recv will disregard cookies unless you are
accessing /admin/::

  if ( !( req.url ~ ^/admin/) ) {
    unset req.http.Cookie;
  }

Quite simple. If, however, you need to do something more complicated,
like removing one out of several cookies, things get
difficult. Unfortunately Varnish doesn't have good tools for
manipulating the Cookies. We have to use regular expressions to do the
work. If you are familiar with regular expressions you'll understand
whats going on. If you don't I suggest you either pick up a book on
the subject, read through the *pcrepattern* man page or read through
one of many online guides.

Let me show you what Varnish Software uses. We use some cookies for
Google Analytics tracking and similar tools. The cookies are all set
and used by Javascript. Varnish and Drupal doesn't need to see those
cookies and since Varnish will cease caching of pages when the client
sends cookies we will discard these unnecessary cookies in VCL. 

In the following VCL we discard all cookies that start with a
underscore::

  // Remove has_js and Google Analytics __* cookies.
  set req.http.Cookie = regsuball(req.http.Cookie, "(^|;\s*)(_[_a-z]+|has_js)=[^;]*", "");
  // Remove a ";" prefix, if present.
  set req.http.Cookie = regsub(req.http.Cookie, "^;\s*", "");

Let me show you an example where we remove everything except the
cookies named COOKIE1 and COOKIE2 and you can marvel at it::

  sub vcl_recv {
    if (req.http.Cookie) {
      set req.http.Cookie = ";" + req.http.Cookie;
      set req.http.Cookie = regsuball(req.http.Cookie, "; +", ";");
      set req.http.Cookie = regsuball(req.http.Cookie, ";(COOKIE1|COOKIE2)=", "; \1=");
      set req.http.Cookie = regsuball(req.http.Cookie, ";[^ ][^;]*", "");
      set req.http.Cookie = regsuball(req.http.Cookie, "^[; ]+|[; ]+$", "");

      if (req.http.Cookie == "") {
          remove req.http.Cookie;
      }
    }
  }

A somewhat simpler example that can accomplish almost the same can be
found below. Instead of filtering out the other cookies it picks out
the one cookie that is needed, copies it to another header and then
copies it back, deleting the original cookie header.::

  sub vcl_recv {
         # save the original cookie header so we can mangle it
        set req.http.X-Varnish-PHP_SID = req.http.Cookie;
        # using a capturing sub pattern, extract the continuous string of 
        # alphanumerics that immediately follows "PHPSESSID="
        set req.http.X-Varnish-PHP_SID = 
           regsuball(req.http.X-Varnish-PHP_SID, ";? ?PHPSESSID=([a-zA-Z0-9]+)( |;| ;).*","\1");
        set req.http.Cookie = req.X-Varnish-PHP_SID;
        remove req.X-Varnish-PHP_SID;
   }   

There are other scary examples of what can be done in VCL in the
Varnish Cache Wiki.


Cookies coming from the backend
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If your backend server sets a cookie using the Set-Cookie header
Varnish will not cache the page in the default configuration.  A
hit-for-pass object (see :ref:`user-guide-vcl_actions`) is created.
So, if the backend server acts silly and sets unwanted cookies just unset
the Set-Cookie header and all should be fine. 



Cache-Control
~~~~~~~~~~~~~

The Cache-Control instructs caches how to handle the content. Varnish
cares about the *max-age* parameter and uses it to calculate the TTL
for an object. 

"Cache-Control: nocache" is ignored but if you need this you can
easily add support for it.

So make sure you issue a Cache-Control header with a max-age
header. You can have a look at what Varnish Software's drupal server
issues::

  $ GET -Used http://www.varnish-software.com/|grep ^Cache-Control
  Cache-Control: public, max-age=600

Age
~~~

Varnish adds an Age header to indicate how long the object has been
kept inside Varnish. You can grep out Age from varnishlog like this::

  varnishlog -i TxHeader -I ^Age

Pragma
~~~~~~

An HTTP 1.0 server might send "Pragma: nocache". Varnish ignores this
header. You could easily add support for this header in VCL.

In vcl_backend_response::

  if (beresp.http.Pragma ~ "nocache") {
        set beresp.uncacheable = true;
	set beresp.ttl = 120s; // how long not to cache this url.
  }

Authorization
~~~~~~~~~~~~~

If Varnish sees an Authorization header it will pass the request. If
this is not what you want you can unset the header.

Overriding the time-to-live (ttl)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes your backend will misbehave. It might, depending on your
setup, be easier to override the ttl in Varnish than to fix your
somewhat cumbersome backend. 

You need VCL to identify the objects you want and then you set the
beresp.ttl to whatever you want::

  sub vcl_backend_response {
      if (req.url ~ "^/legacy_broken_cms/") {
          set beresp.ttl = 5d;
      }
  }

The example will set the TTL to 5 days for the old legacy stuff on
your site.

Forcing caching for certain requests and certain responses
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Since you still have this cumbersome backend that isn't very friendly
to work with you might want to override more stuff in Varnish. We
recommend that you rely as much as you can on the default caching
rules. It is perfectly easy to force Varnish to lookup an object in
the cache but it isn't really recommended.


Normalizing your namespace
~~~~~~~~~~~~~~~~~~~~~~~~~~

Some sites are accessed via lots of
hostnames. http://www.varnish-software.com/,
http://varnish-software.com/ and http://varnishsoftware.com/ all point
at the same site. Since Varnish doesn't know they are different,
Varnish will cache different versions of every page for every
hostname. You can mitigate this in your web server configuration by
setting up redirects or by using the following VCL::

  if (req.http.host ~ "(?i)^(www.)?varnish-?software.com") {
    set req.http.host = "varnish-software.com";
  }


.. _users-guide-vary:

HTTP Vary
---------

*HTTP Vary is not a trivial concept. It is by far the most
misunderstood HTTP header.*

A lot of the response headers tell the client something about the HTTP
object being delivered. Clients can request different variants a an
HTTP object, based on their preference. Their preferences might cover
stuff like encoding or language. When a client prefers UK English this
is indicated through "Accept-Language: en-uk". Caches need to keep
these different variants apart and this is done through the HTTP
response header "Vary".

When a backend server issues a "Vary: Accept-Language" it tells
Varnish that its needs to cache a separate version for every different
Accept-Language that is coming from the clients.

If two clients say they accept the languages "en-us, en-uk" and "da,
de" respectively, Varnish will cache and serve two different versions
of the page if the backend indicated that Varnish needs to vary on the
Accept-Language header.

Please note that the headers that Vary refer to need to match
*exactly* for there to be a match. So Varnish will keep two copies of
a page if one of them was created for "en-us, en-uk" and the other for
"en-us,en-uk". Just the lack of space will force Varnish to cache
another version.

To achieve a high hitrate whilst using Vary is there therefor crucial
to normalize the headers the backends varies on. Remember, just a
difference in case can force different cache entries.

The following VCL code will normalize the Accept-Language headers, to
one of either "en","de" or "fr"::

    if (req.http.Accept-Language) {
        if (req.http.Accept-Language ~ "en") {
            set req.http.Accept-Language = "en";
        } elsif (req.http.Accept-Language ~ "de") {
            set req.http.Accept-Language = "de";
        } elsif (req.http.Accept-Language ~ "fr") {
            set req.http.Accept-Language = "fr";
        } else {
            # unknown language. Remove the accept-language header and 
	    # use the backend default.
            remove req.http.Accept-Language
        }
    }

The code sets the Accept-Encoding header from the client to either
gzip, deflate with a preference for gzip.

Vary parse errors
~~~~~~~~~~~~~~~~~

Varnish will return a 503 internal server error page when it fails to
parse the Vary server header, or if any of the client headers listed
in the Vary header exceeds the limit of 65k characters. An SLT_Error
log entry is added in these cases.

Pitfall - Vary: User-Agent
~~~~~~~~~~~~~~~~~~~~~~~~~~

Some applications or application servers send *Vary: User-Agent* along
with their content. This instructs Varnish to cache a separate copy
for every variation of User-Agent there is. There are plenty. Even a
single patchlevel of the same browser will generate at least 10
different User-Agent headers based just on what operating system they
are running. 

So if you *really* need to Vary based on User-Agent be sure to
normalize the header or your hit rate will suffer badly. Use the above
code as a template.

