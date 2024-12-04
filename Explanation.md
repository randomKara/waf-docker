
# Objective

Create a WAF (Web Application Firewall) between a third-party application and the user. This WAF must be integrated as a "reverse proxy". 

# How to test
For the third-party application we'll use [DVWA](https://github.com/digininja/DVWA) (Damn Vulnerable Web Application). This application has all the most common major security breaches that we can find in a web application. 

To start it on the port 80:
```bash
docker run -p 80:80 --name dvwa vulnerables/web-dvwa:latest
```
The URL to access it: http://localhost:80

 For the WAF part, we'll use [owasp-modsecurity/ModSecurity](https://github.com/owasp-modsecurity/ModSecurity). The aim is to integrate it with an Apache Docker under Alpine. 
Docker for ease of use, Apache rather than NGINX for security reasons, and Alpine for its lightness. 
A docker accessible from docker.io already exists and can be used in this way:

```bash
docker run -d \
	--name modsecurity \
	-p 8080:8080 \
	-p 8443:8443 \
	-e BACKEND=http://dvwa:80 \
	-e SERVER_NAME=localhost   \
	--link dvwa:dvwa \
	owasp/modsecurity-crs:apache-alpine
```

| Argument                            | Signifaction            | Explanation                                                                                                                                                  |
| ----------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| --name                              | name given              | Name Given to the docker.                                                                                                                                    |
| -p                                  | port use                | Port use by the application (/!\ dont use the same as DVWA.                                                                                                  |
| -e BACKEND                          | env variable definition | Where the proxy redirect                                                                                                                                     |
| -e SERVER_NAME                      | env variable definition | Where we want to access the application. Here it means if we go http://localhost:8080 we'll go to DVWA through the reverse proxy                             |
| --link                              | network link            | How we want to connect our application to reverse proxy. Here it means dvwa (the docker) will be accessible to http://dvwa **INSIDE** the modsecurity docker |
| owasp/modsecurity-crs:apache-alpine | image to use            | What image to pull (here, [this image](https://hub.docker.com/r/owasp/modsecurity-crs/))                                                                     |
Note: strangely, we can't pull docker image if we don't have the web page opened.

# Major issue
The image looks like this
```bash
user ~: docker pull owasp/modsecurity-crs:apache-alpine

user ~: docker image ls
REPOSITORY              TAG             IMAGE ID       CREATED          SIZE
owasp/modsecurity-crs   apache-alpine   10af986c210c   44 hours ago     103MB
```
It is far to heavy for what we need. Let's look at the [official](https://github.com/coreruleset/modsecurity-crs-docker) [Dockerfile](https://github.com/coreruleset/modsecurity-crs-docker/blob/main/apache/Dockerfile-alpine) of this image. 

We can see how it is build:
```Dockerfile
FROM httpd:${HTTPD_VERSION}-alpine AS build
```
It uses the official httpd alpine image that we can found [here](https://github.com/docker-library/httpd/blob/3056c115a9f3c2467cc6f67470cfded70c4adc64/2.4/alpine/Dockerfile). When we look at it, we can see it compile Apache from an official source instead of just downloading it. To permit the compilation, the docker must download a large number of packets as shown in the code bellow:
```Dockerfile
RUN set -eux; \
	\
	apk add --no-cache --virtual .build-deps \
		apr-dev \
		apr-util-dev \
		coreutils \
		dpkg-dev dpkg \
		gcc \
		gnupg \
		libc-dev \
		patch \
		# mod_md
		curl-dev \
		jansson-dev \
		# mod_proxy_html mod_xml2enc
		libxml2-dev \
		# mod_lua
		lua-dev \
		make \
		# mod_http2
		nghttp2-dev \
		# mod_session_crypto
		openssl \
		openssl-dev \
		pcre-dev \
		tar \
		# mod_deflate
		zlib-dev \
		# mod_brotli
		brotli-dev \
	; \
	\
```
The issue is that these packages aren't used by Apache once it has been compiled, so they expose the docker to more security breach, given that owasp/modsecurity-crs:apache-alpine inherits this Dockerfile.

An other issue is how the modsecurity's Dockerfile is write. For example:
```Dockerfile
FROM httpd:${HTTPD_VERSION}-alpine AS build

#...

RUN set -eux; \
    apk add --no-cache \
        ca-certificates \
        curl \
        curl-dev \
        gnupg; \
    mkdir /opt/owasp-crs; \
    curl -sSL https://github.com/coreruleset/coreruleset/releases/download/v${CRS_RELEASE}/coreruleset-${CRS_RELEASE}-minimal.tar.gz -o v${CRS_RELEASE}-minimal.tar.gz; \
    curl -sSL https://github.com/coreruleset/coreruleset/releases/download/v${CRS_RELEASE}/coreruleset-${CRS_RELEASE}-minimal.tar.gz.asc -o coreruleset-${CRS_RELEASE}-minimal.tar.gz.asc; \
    gpg --fetch-key https://coreruleset.org/security.asc; \
    gpg --verify coreruleset-${CRS_RELEASE}-minimal.tar.gz.asc v${CRS_RELEASE}-minimal.tar.gz; \
    tar -zxf v${CRS_RELEASE}-minimal.tar.gz --strip-components=1 -C /opt/owasp-crs; \
    rm -f v${CRS_RELEASE}-minimal.tar.gz coreruleset-${CRS_RELEASE}-minimal.tar.gz.asc; \
    mv -v /opt/owasp-crs/crs-setup.conf.example /opt/owasp-crs/crs-setup.conf

```
Their is many undefined variable (HTTPD_VERSION and CRS_RELEASE is this example). They're defined in [this](https://github.com/coreruleset/modsecurity-crs-docker/blob/main/docker-bake.hcl) file. We'll have to either create a script to replace every undefined variable or do it manually (I chose the second solution).
# Solution
Their is two solutions to this issue. First we can do our own httpd alpine image and create the modsecurity docker with it, or we can just edit the last stage of the modsecurity's Dockerfile. I chose the second solution, because building httpd alpine would have meant to edit every single stage of the modsecurity's Dockerfile.

I still created my own the httpd alpine because I didn't wanted the solution to pull the official image each time it's updated or deployed on a new environment. Their is still an error the official httpd alpine image. When we download the projet, everything is correct, but when we move the element in a other directory for example, their is an error "permission refused" on the httpd-foreground file. To avoid that, we need to add an chmod at the end of the httpd-local alpine image:
```Dockerfile
COPY httpd/httpd-foreground /usr/local/bin/

RUN chmod 777 /usr/local/bin/httpd-foreground
```

So I edited the last stage of modescurity's Dockerfile like this:
```Dockerfile
FROM alpine:3.20

ARG MODSEC2_VERSION="2.9.8"
ARG LUA_VERSION="5.3"
ARG LUA_MODULES=["lua-lzlib","lua-socket"]

RUN apk add --no-cache \
    apr \
    apr-util \
    ca-certificates \ 
    pcre  \
    libxml2 \
    lua${LUA_VERSION} \
    lua-socket \
    lua-lzlib \
    openssl \
    curl \
    libfuzzy2 \
    sed \
    yajl && \
    rm -rf /var/cache/apk/* /tmp/*
    
RUN adduser -u 82 -D -S -G www-data www-data

ENV HTTPD_PREFIX=/usr/local/apache2
ENV PATH=$HTTPD_PREFIX/bin:$PATH
RUN mkdir -p $HTTPD_PREFIX && chown -R www-data:www-data $HTTPD_PREFIX

COPY --from=local-httpd:latest /usr/local/apache2/ /usr/local/apache2/

# -------------------------------------------------------
# ------------------- END HTTPD PART --------------------
# -------------------------------------------------------

LABEL maintainer="Felipe Zipitria <felipe.zipitria@owasp.org>"

#ENV ... 

COPY --from=build /usr/local/apache2/modules/mod_security2.so /usr/local/apache2/modules/mod_security2.so
COPY --from=build /usr/local/apache2/ModSecurity-${MODSEC2_VERSION}/unicode.mapping /etc/modsecurity.d/unicode.mapping
COPY --from=crs_release /opt/owasp-crs /opt/owasp-crs
COPY src/etc/modsecurity.d/*.conf /etc/modsecurity.d/
COPY src/bin/* /usr/local/bin/
COPY src/opt/modsecurity/activate-*.sh /opt/modsecurity/
COPY apache/conf/ /usr/local/apache2/conf/
COPY apache/docker-entrypoint.sh /


RUN addgroup -S httpd && adduser -SH httpd httpd

RUN set -eux; \
    ln -sv /opt/owasp-crs /etc/modsecurity.d/; \
    mkdir -p /var/log/apache2; \
    mkdir -p /tmp/modsecurity/data; \
    mkdir -p /tmp/modsecurity/upload; \
    mkdir -p /tmp/modsecurity/tmp; \
    chown -R httpd:httpd \
        /var/log/ \
        /usr/local/apache2/ \
        /etc/modsecurity.d \
        /tmp/modsecurity \
        /opt/owasp-crs

USER httpd

HEALTHCHECK CMD /usr/local/bin/healthcheck

ENTRYPOINT ["/docker-entrypoint.sh"]

# Define the startup command
CMD ["/usr/local/apache2/bin/httpd", "-D", "FOREGROUND"]
```
In the original Dockerfile, in the last "RUN, their are some "sed" commands that I removed. They were meant to reconfigure /conf/* file. Reconfigure configuration file seems a bit overdone.

# Final
The final docker reverse proxy weighs 48.5MB. Here were it is ditrbuted:

File > 1MB
```bash
-rwxr-xr-x    1 root     root        4.3M Oct 20 07:02 /lib/libcrypto.so.3  
-rwxr-xr-x    1 root     root        1.0M May 18  2024 /usr/lib/libxml2.so.2.12.7  
-rwxr-xr-x    1 root     root        1.6M Feb 29  2024 /usr/lib/libunistring.so.5.1.0  
-r--r--r--    1 httpd    httpd       2.6M Dec  4 08:50 /usr/local/apache2/modules/mod_security2.
```

```bash
/ $ du -h -d 1 / #disk usage - human reading - depht = 1 - root file  
4.0K    ./mnt  
4.0K    ./run  
924.0K  ./bin  
6.1M    ./lib  
20.0K   ./tmp  
4.0K    ./srv  
8.0K    ./home  
16.0K   ./media  
80.0K   ./sbin  
22.5M   ./usr  
31.9M   .
```

```bash
/ $ du -h -d 1 /usr/ #disk usage - human reading - depth = 1 - /usr/ file  
1.3M    /usr/bin  
13.1M   /usr/local  
7.2M    /usr/lib  
832.0K  /usr/share  
20.0K   /usr/sbin  
22.5M   /usr/
```

```bash
/ $ apk info -v #all installed package  
WARNING: opening from cache [https://dl-cdn.alpinelinux.org/alpine/v3.20/main](https://dl-cdn.alpinelinux.org/alpine/v3.20/main): No such file or directory  
WARNING: opening from cache [https://dl-cdn.alpinelinux.org/alpine/v3.20/community](https://dl-cdn.alpinelinux.org/alpine/v3.20/community): No such file or directory  
alpine-baselayout-3.6.5-r0  
alpine-baselayout-data-3.6.5-r0  
alpine-keys-2.4-r1  
apk-tools-2.14.4-r0  
apr-1.7.5-r0  
apr-util-1.6.3-r1  
brotli-libs-1.1.0-r2  
busybox-1.36.1-r29  
busybox-binsh-1.36.1-r29  
c-ares-1.33.1-r0  
ca-certificates-20240705-r0  
ca-certificates-bundle-20240705-r0  
curl-8.11.0-r2  
libcrypto3-3.3.2-r1  
libcurl-8.11.0-r2  
libexpat-2.6.4-r0  
libfuzzy2-2.14.1-r2  
libidn2-2.3.7-r0  
libpsl-0.21.5-r1  
libssl3-3.3.2-r1  
libunistring-1.2-r0  
libuuid-2.40.1-r1  
libxml2-2.12.7-r0  
linenoise-1.0-r5  
lua-lzlib-0.4.3-r2  
lua-socket-3.1.0-r1  
lua5.3-5.3.6-r6  
lua5.3-libs-5.3.6-r6  
lua5.3-lzlib-0.4.3-r2  
lua5.3-socket-3.1.0-r1  
musl-1.2.5-r0  
musl-utils-1.2.5-r0  
nghttp2-libs-1.62.1-r0  
openssl-3.3.2-r1  
pcre-8.45-r3  
scanelf-1.3.7-r2  
sed-4.9-r2  
ssl_client-1.36.1-r29  
xz-libs-5.6.2-r0  
yajl-2.1.0-r9  
zlib-1.3.1-r1  
zstd-libs-1.5.6-r0
```