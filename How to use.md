
# Requirement
You'll need to install: docker, docker-compose, and the project.

# How to build
First you'll need to build the local-httpd:alpine. Go to the alpine_apache directory.
Then:
```bash
user ~/WAF/alpine_apache: docker build \
	--no-cache \
	-t local-httpd:alpine \
	-f httpd/Dockerfile .
```

| Argument   | Signification           | Explanation                                                                                        |
| ---------- | ----------------------- | -------------------------------------------------------------------------------------------------- |
| --no-cache | Do not use cased docker | For a clean installation, you need to build the entire docker without using previous docker build. |
| -t         | Name gave to the image  | The name is important as you'll call this image in the modsecurity's Dockerfile                    |
| -f         | File to use             | What file to use. Here we'll use the file named Dockerfile in the directory httpd/                 |

Then you can build the main docker:
```bash
user ~/WAF/alpine_apache: docker build \
	--no-cache \
	-t modsecurity:latest \
	-f apache/Dockerfile-alpine-light .
```

You'll need to launch the application to be protected. Here we lunch DVWA on the port 80:
```bash
user ~/WAF/alpine_apache: docker run \
	-p 80:80 \
	 --name dvwa \
	vulnerables/web-dvwa:latest
```

Then you can start the reverse proxy. 
```bash
user ~/WAF/alpine_apache: docker run -d \
	--rm \
	--name modsecurity \
	-p 8080:8080 \
	-p 8443:8443 \
	-e BACKEND=http://dvwa:80 \
	-e SERVER_NAME=localhost \
	--link dvwa:dvwa \
	modsecurity:latest
```

| Argument           | Signifaction            | Explanation                                                                                                                                                  |
| ------------------ | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| -d                 | detach                  | Run container in background and print container ID                                                                                                           |
| --name             | name given              | Assign a name to the container                                                                                                                               |
| -p                 | port use                | Port(s) to use by the application (/!\ don't use the same as DVWA)                                                                                           |
| -e BACKEND         | env variable definition | Where the proxy redirect                                                                                                                                     |
| -e SERVER_NAME     | env variable definition | Where we want to access the application. Here it means if we go http://localhost:8080 we'll go to DVWA through the reverse proxy                             |
| --link             | network link            | How we want to connect our application to reverse proxy. Here it means dvwa (the docker) will be accessible to http://dvwa **INSIDE** the modsecurity docker |
| modsecurity:latest | image to use            | What image to use.                                                                                                                                           |

