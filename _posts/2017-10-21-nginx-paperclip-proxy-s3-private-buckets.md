---
layout: post
title:  Nginx & Paperclip & S3 private buckets
date:   2017-10-21
categories: nginx s3 cloud deployment
---

First configure Paperclip to treat S3 bucket as private (by default it's
public) somewhere in your initrializers

```ruby
Paperclip::Attachment.default_options[:s3_permissions] = :private
```

Then I have simple helper, not to have to repeat the code, as downloads can
be triggered on multiple placies through the applications.

```ruby
def download(url)
  ct = Rack::Mime::MIME_TYPES.fetch(File.extname(url), 'application/octet-stream')

  response.headers['X-Accel-Redirect'] = '/reproxy'
  response.headers['X-Reproxy-URL'] = url
  response.headers['Content-Type'] = ct

  render plain: ''
end
```

The helper is called from somwhere in controllers

```ruby
download(attachment.file.expiring_url(10))
```

Paperclip will generated unique URL that will be valid for 10s and that nginx
can use to "high-jack" the response.

And finally the configuration of Nginx

```
location /reproxy {
    internal;
    resolver               8.8.8.8;
    set $reproxy           $upstream_http_x_reproxy_url;
    proxy_pass             $reproxy;
    proxy_hide_header      Content-Type;
    proxy_http_version     1.1;
    proxy_set_header       Connection "";
    proxy_set_header       Authorization '';
    proxy_hide_header      x-amz-id-2;
    proxy_hide_header      x-amz-request-id;
    proxy_hide_header      x-amz-meta-server-side-encryption;
    proxy_hide_header      x-amz-server-side-encryption;
    proxy_hide_header      Set-Cookie;
    proxy_ignore_headers   Set-Cookie;
    proxy_intercept_errors on;
}
```

The idea is based on [this](http://ls4.sourceforge.net/doc/howto/ddt.html)
documentation for LS4, plus compilation of several
[StackOverflow](https://stackoverflow.com) answers to make the "/reproxy"
target work correctly with S3 authorization.
