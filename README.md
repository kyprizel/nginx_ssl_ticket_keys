This functionality is already in Nginx dev, don't use this path!
================================================================

nginx_ssl_ticket_keys
=====================

Patch for nginx which adds external keyfile support for [RFC5077](http://tools.ietf.org/html/rfc5077) TLS extension.

Directives
==========

ssl_session_ticket_key
----------------------
**syntax:** *ssl_session_ticket_key &lt;string&gt;*

**default:** *none*

**context:** *server*

Sets ticket key file.

Key file format and how it works
================================

File can contain multiple keys.

    -----BEGIN SESSION TICKET KEY-----
    MjAxMy0xMC0wMTowMDowMEFBQUFBQUFBQUFBQUFBQUFCQkJCQkJCQkJCQkJCQkJC
    -----END SESSION TICKET KEY-----
    -----BEGIN SESSION TICKET KEY-----
    QUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFBQUFB
    -----END SESSION TICKET KEY-----

Inside there are base64 encoded 48 byte structures, like:

    NAME[16] = '2013-10-01:00:00'
    AES_KEY[16] = 'AAAAAAAAAAAAAAAA'
    HMAC_KEY[16] = 'BBBBBBBBBBBBBBBB'

*NAME* identificates the key, used for session resumption.

For example:

If you need to accept sessions for users with previous keys and deploy a new one, just add it in the begining of the key file.
Then all the previous sessions will be accepted, but new tickets will be reissued using the new key (first key from the list).

**NB: You need to manually update ticket keys to make them expire!**

Oneliner for key rotation, generates new key and stores not more than X=2 old keys:

    $ openssl rand -base64 48 | awk '{print "-----BEGIN SESSION TICKET KEY-----"; print; print "-----END SESSION TICKET KEY-----"}' >> ticket.key.new && cat ticket.key | awk 'sa==1{n++;sa=1}/-----BEGIN SESSION TICKET KEY-----/{sa=1;X=2}{if(n<3*X){print;}}' >> ticket.key.new && mv ticket.key.new ticket.key


Installation
============

Grab the nginx source code from [nginx.org](<http://nginx.org/>).
Patch and compile.

    wget 'http://nginx.org/download/nginx-VERSION.tar.gz'
    tar -xzvf nginx-VERSION.tar.gz
    patch -p0 < nginx-VERSION.patch

    ./configure --with-debug --with-http_ssl_module
    make
    make install

Example configuration
=====================

    server {
        listen       443;
        server_name  localhost;

        ssl                  on;
        ssl_certificate      ssl/testhost.crt;
        ssl_certificate_key  ssl/testhost.key;

        ssl_protocols  SSLv3 TLSv1 TLSv1.1 TLSv1.2;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers   on;

        ssl_session_ticket_key sst/ticket.key;

        location / {
            root   html;
        }
    }

Testing
=======

There is an [awesome tool](https://github.com/vincentbernat/rfc5077) for testing RFC5077 TLS extension support.
