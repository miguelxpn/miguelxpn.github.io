---
layout: post
title:  "How I found and fixed a bug in PHP's standard library"
date:   2020-03-01 02:06:14 -0300
categories: coding php opensource
---
I’ve been a software developer at EBANX since 2016. There we have a mechanism called `Release Candidate`. Every time we deploy something new to our monolith PHP system we end up deploying it first to the release candidate server. Then we elect a few random requests and proxy it from our live normal servers to the release candidate server, if those requests go well with no errors then the deploy occurs in the normal API servers, if an error occurs the proxying mechanism is disabled and the responsible for the code can better debug the problem in the release candidate server and rollback the code if needed.

I was deploying some code once and I noticed something weird in the logs. Some requests had two Host headers once they arrived in the release candidate server. Normally we simply copy the headers that came from the original request, so the request should arrive in the release candidate with the host header containing the url of the original server. In the cases where two Hosts were present the top one was the url of the release candidate server and the second one was the original one containing the original api server. This puzzled me, the code I was modifying had nothing to do with the proxying mechanism, and after looking at older logs I found out that this always occurred a few times. Since this mechanism is critical to the safety of our deploys I dropped everything and started to investigate.

Naturally I assumed that the bug was somewhere in our proxying code, after digging for a while I couldn’t find anything wrong with the code. This is the function we use to rewrite the Header of the request to the release candidate with the original host:

{% highlight php %}
private static function adjustRequestTarget(RequestInterface $request): RequestInterface {
  $original_host = $request->getHeaderLine('Host');
  $new_uri = $request->getUri()->withHost(static::getReleaseCandidateInstanceHost());
  return $request
    ->withUri($new_uri)
    ->withHeader('Host', $original_host);
}
{% endhighlight %}

We are using Guzzle as the http client. Internally the headers are represented as an array inside the class GuzzleHttp\Psr7\Request. I noticed that when calling the withHeader function the Host header is shifted to the end of array, but regardless of this detail I couldn’t reproduce the issue when testing locally with the exact same requests that caused the problem in production.

I ended up noticing something pretty interesting. The `stream` option being passed during the request was different on my tests than the production code, this made guzzle use a different http handler, with `stream` set as true it uses the PHP stream handler, with it set as `false` it uses cURL, I was testing it with false, changing it to true made me able to reproduce the issue when replaying a request that caused the problem in production, now I just needed to drill down enough in the code that was called to determine what was causing the problem.

After cloning the PHP source and reading for a while I finally found the culprit, the problem is caused by a combination of the change of header array order that the previously referenced `adjustRequestTarget` function causes, and the way the http fopen wrapper handles the http headers.

Inside the http_fopen_wrapper.c code we have these two interesting snippets:

```c
if ((s = strstr(t, "host:")) &&
                (s == t || *(s-1) == '\r' || *(s-1) == '\n' ||
                             *(s-1) == '\t' || *(s-1) == ' ')) {
                 have_header |= HTTP_HEADER_HOST;
            }
```

```c
/* Send Host: header so name-based virtual hosts work */
if ((have_header & HTTP_HEADER_HOST) == 0) {
    smart_str_appends(&req_buf, "Host: ");
    smart_str_appends(&req_buf, ZSTR_VAL(resource->host));
    if ((use_ssl && resource->port != 443 && resource->port != 0) ||
        (!use_ssl && resource->port != 80 && resource->port != 0)) {
        smart_str_appendc(&req_buf, ':');
        smart_str_append_unsigned(&req_buf, resource->port);
    }
    smart_str_appends(&req_buf, "\r\n");
}
```

The first snippet searches in the header string for an instance of the `host:` string in the beginning of a line, if it finds one there then it enables the have_header `HTTP_HEADER_HOST` flag, later the second snippet checks that flag and if it’s not enabled then it appends a Host header with the target address.

Turns out that all the affected requests had a header with the string `host:` somewhere in them before the Host header (which was shifted to the end of array due to us changing it back to the original host), this made the check fail and the wrapper thought we didn’t have a host header because strstr only returns the first instance of a string. So the wrapper ended up appending a new Host header with the target url and it still sends the one that we were trying to send originally causing the issue.

The following test script:

```php
<?php
$opts = array(
  'http'=>array(
    'method'=>"GET",
    'header'=>"RandomHeader: foobar\r\n" .
              "Cookie: foo=bar\r\n" .
              "Host: c764dd43.ngrok.io  \r\n"
  )
);
$context = stream_context_create($opts);
$fp = fopen('http://c764dd43.ngrok.io', 'r', false, $context);
fpassthru($fp);
```

Will generate the following raw headers:

```
GET / HTTP/1.0
RandomHeader: foobar
Cookie: foo=bar
Host: c764dd43.ngrok.io
X-Forwarded-For: 2804:14c:8781:8c3a:3805:2fbe:1845:15e6
```

Now if we change the RandomHeader value to something that contains the `host:` string:

```php
<?php
$opts = array(
  'http'=>array(
    'method'=>"GET",
    'header'=>"RandomHeader: localhost:8080\r\n" .
              "Cookie: foo=bar\r\n" .
              "Host: testcustomheader  \r\n"
  )
);
$context = stream_context_create($opts);
$fp = fopen('http://c764dd43.ngrok.io', 'r', false, $context);
fpassthru($fp); (edited)
```

We'll get two `Host` headers in the raw request:

```
GET / HTTP/1.0
Host: c764dd43.ngrok.io
RandomHeader: localhost:8080
Cookie: foo=bar
Host: testcustomhostheader
X-Forwarded-For: 2804:14c:8781:8c3a:3805:2fbe:1845:15e6
```

According to the [RFC 7320][rfc-7320] a server must respond with a 400 (bad request) to requests that contain more than one Host header, so every request affected by this bug should break on a properly implemented server.

With this information I decided to open a [bug][bug] on the PHP issue tracker. After a few days of inactivity I decided I had enough knowledge to try to implement a fix myself!

I cloned the PHP git repository from github and compiled it from the master branch.

I changed the snippet that checks if the host header is present to check for every occurence of the string:

```c
while ((s = strstr(s, "host:"))) {
	if (s == t || *(s-1) == '\r' || *(s-1) == '\n' ||
	                 *(s-1) == '\t' || *(s-1) == ' ') {
		 have_header |= HTTP_HEADER_HOST;
	}
	s++;
}
```

I recompiled, ran the test script again and it worked! Now I needed to find out the project contributing guidelines.

I found out that PHP has a test suite composed by `phpt` files, these files are a mixture of plaintext and php with different sections. The most important sections are `FILE` and `EXPECTED`, inside the file section there should be a PHP script that should output something to stdout. The `EXPECTED` section contains the expected output, the test script then compares the output produced by the script inside FILE to the output on the `EXPECTED` section, if they match the test passes, if they don't then it fails. It's very simple and effective. I changed the script I was using to reproduce the issue to become a .phpt file so we could have automated tests for this bug:

```php
--TEST--
Bug #79265 (Improper injection of Host header when using fopen for http requests)
--INI--
allow_url_fopen=1
--SKIPIF--

--FILE--
<?php
require 'server.inc';

$responses = array(
    "data://text/plain,HTTP/1.0 200 OK\r\n\r\n",
);

$pid = http_server("tcp://127.0.0.1:12342", $responses, $output);

$opts = array(
  'http'=>array(
    'method'=>"GET",
    'header'=>"RandomHeader: localhost:8080\r\n" .
              "Cookie: foo=bar\r\n" .
              "Host: userspecifiedvalue\r\n"
  )
);
$context = stream_context_create($opts);
$fd = fopen('http://127.0.0.1:12342/', 'rb', false, $context);
fseek($output, 0, SEEK_SET);
echo stream_get_contents($output);
fclose($fd);

http_server_kill($pid);

?>
--EXPECT--
GET / HTTP/1.0
Connection: close
RandomHeader: localhost:8080
Cookie: foo=bar
Host: userspecifiedvalue

```

I commited the fix alongside with the test script and opened a [pull request][pull-request]. It was merged a few hours later.

[rfc-7320]: https://tools.ietf.org/html/rfc7230#section-5.4
[bug]: https://bugs.php.net/bug.php?id=79265
[pull-request]: https://github.com/php/php-src/pull/5201
