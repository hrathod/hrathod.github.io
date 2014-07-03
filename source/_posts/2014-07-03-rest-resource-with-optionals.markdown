---
layout: post
title: "REST resource with Optionals"
date: 2014-07-03 10:44:30 -0400
comments: true
categories: ["REST", "RESTEasy", "Java8"]
---
One of the new classes added to Java 8 is
[java.util.Optional](http://docs.oracle.com/javase/8/docs/api/java/util/Optional.html "java.util.Optional Javadocs"),
intended to reduce the errors surrounding the use of _null_.  One of the
recommended uses of the Optional class is the return value of a method that may
return a _null_ value.

As an extension of this idea, it might be an interesting experiment to use
Optional return values from RESTful service resource methods, to indicate
when a resource may return a _null_.  A common scenario might be to pass
an ID to a resource method, which then looks up a resource by that ID.  What
if that resource does not exist?  Normally, the method would return _null_.
Instead, to document the fact that the method may return null, the Optional
class can be used.

<!-- more -->

The question becomes, then, how do we map Optional return values to correct HTTP
status codes?  The following experiment is one way to approach the problem.

Let's begin with the basic [RESTEasy](http://resteasy.jboss.org/ "RESTEasy Website")
web.xml setup.


``` xml web.xml
<!DOCTYPE web-app PUBLIC
		"-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
		"http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
	<display-name>Example Webapp with RESTEasy and Optionals</display-name>
	<servlet>
		<servlet-name>resteasy</servlet-name>
		<servlet-class>org.jboss.resteasy.plugins.server.servlet.HttpServletDispatcher</servlet-class>
		<init-param>
			<param-name>javax.ws.rs.Application</param-name>
			<param-value>io.github.hrathod.blog.resteasy.SimpleApplication</param-value>
		</init-param>
	</servlet>
	<servlet-mapping>
		<servlet-name>resteasy</servlet-name>
		<url-pattern>/*</url-pattern>
	</servlet-mapping>
</web-app>
```

We simply set up the RESTEasy dispatch servlet, and configure it to use our
Application subclass.  The Application subclass in turn will configure RESTEasy
to use the various classes needed for setting up the services.

``` java SimpleApplication.java
import java.util.HashSet;
import java.util.Set;

import javax.ws.rs.core.Application;

public class SimpleApplication extends Application {

	private final Set<Object> singletons = new HashSet<>();

	public SimpleApplication () {
		this.singletons.add(new SimpleResource());
		this.singletons.add(new OptionalFilter());
	}

	@Override
	public Set<Object> getSingletons() {
		return this.singletons;
	}
}
```

The SimpleApplication class simply registers two classes, a resource class
and a container filter.  The SimpleResource class is just a regular resource
class, with the normal annotations.

``` java SimpleResource.java
import java.util.Optional;

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/simple")
public class SimpleResource {

	private final SimpleDao dao = new SimpleDao();

	@GET
	@Produces(MediaType.TEXT_PLAIN)
	@Path("/{id}")
	public Optional<SimpleModel> get (@PathParam("id") String id) {
		return this.dao.get(id);
	}

}
```

The only interesting part of the SimpleResource class is that it returns an
Optional<SimpleModel> instead of simply returning a SimpleModel.  This is
nice documentation within the code that the method may return a response
with a _null_ value.  Also notice that the DAO is returning an Optional
as well, which also indicates that a _null_ may be returned if the
given id is not found.

The (slightly) more interesting class is OptionalFilter.  This class
converts an Optional return value from a resource method to the
appropriate HTTP status code and returns the correct entity.

``` java OptionalFilter.java
import java.io.IOException;
import java.util.Optional;

import javax.ws.rs.container.ContainerRequestContext;
import javax.ws.rs.container.ContainerResponseContext;
import javax.ws.rs.container.ContainerResponseFilter;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.Provider;

import org.jboss.resteasy.annotations.interception.ServerInterceptor;

@Provider
@ServerInterceptor
public class OptionalFilter implements ContainerResponseFilter {
	@Override
	public void filter(ContainerRequestContext requestContext, ContainerResponseContext responseContext) throws IOException {
		Object entity = responseContext.getEntity();
		if (Optional.class.isAssignableFrom(entity.getClass())) {
			Optional<?> optional = (Optional<?>) entity;
			if (optional.isPresent()) {
				responseContext.setEntity(optional.get());
			} else {
				responseContext.setEntity(null);
				responseContext.setStatus(Response.Status.NO_CONTENT.getStatusCode());
			}
		}
	}
}
```

The filter simply checks if the entity returned by the resource method is an
Optional.  If it is, the value is fetched from the Optional if it is present,
and set in the response.  If the value is not present, we alter the HTTP
status code of the response to a "204 No Content", and set the entity to _null_.
The setting of the entity to _null_ may not be entirely necessary, but since
"No Content" implies lack of content, it may be better to ensure the lack of
content.

A Maven project containing a working example is available from the
[GitHub repo](https://github.com/hrathod/resteasy-optionals "RESTEasy-Optionals"].
To run it, clone the repo and use the jetty:run maven goal.

``` bash
$ mvn jetty:run
```

Curl or Wget can be used to test the HTTP status codes.  If the ID is "test", the
content should be delivered.  With any other ID, a "No Content" status should be
returned.

``` bash
$ curl -v http://localhost:8080/simple/test
* Adding handle: conn: 0x1b868b0
* Adding handle: send: 0
* Adding handle: recv: 0
* Curl_addHandleToPipeline: length: 1
* - Conn 0 (0x1b868b0) send_pipe: 1, recv_pipe: 0
* About to connect() to localhost port 8080 (#0)
*   Trying ::1...
* Connected to localhost (::1) port 8080 (#0)
> GET /simple/test HTTP/1.1
> User-Agent: curl/7.32.0
> Host: localhost:8080
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Thu, 03 Jul 2014 21:32:48 GMT
< Content-Type: text/plain
< Content-Length: 10
* Server Jetty(9.2.0.RC0) is not blacklisted
< Server: Jetty(9.2.0.RC0)
<
* Connection #0 to host localhost left intact
test model
```

``` bash
$ curl -v http://localhost:8080/simple/foo
* Adding handle: conn: 0x20a08b0
* Adding handle: send: 0
* Adding handle: recv: 0
* Curl_addHandleToPipeline: length: 1
* - Conn 0 (0x20a08b0) send_pipe: 1, recv_pipe: 0
* About to connect() to localhost port 8080 (#0)
*   Trying ::1...
* Connected to localhost (::1) port 8080 (#0)
> GET /simple/foo HTTP/1.1
> User-Agent: curl/7.32.0
> Host: localhost:8080
> Accept: */*
>
< HTTP/1.1 204 No Content
< Date: Thu, 03 Jul 2014 21:35:37 GMT
< Content-Type: text/plain
* Server Jetty(9.2.0.RC0) is not blacklisted
< Server: Jetty(9.2.0.RC0)
<
* Connection #0 to host localhost left intact
```
