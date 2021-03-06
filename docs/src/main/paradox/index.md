# Play Scala gRPC Example

This example application shows how to use Akka gRPC to both expose and use gRPC services inside an Play application.

The [Play Framework](https://www.playframework.com/) combines productivity and performance making it easy to build 
scalable web applications with Java and Scala. Play is developer friendly with a "just hit refresh" workflow and 
built-in testing support. With Play, applications scale predictably due to a stateless and non-blocking architecture.

[Akka gRPC](https://developer.lightbend.com/docs/akka-grpc/current/overview.html) is a toolkit for building streaming 
gRPC servers and clients on top of Akka Streams.

For detailed documentation refer to https://www.playframework.com/documentation/latest/Home and https://developer.lightbend.com/docs/akka-grpc/current/.

## Obtaining this example

You may download the code from [GitHub](https://github.com/playframework/play-scala-grpc-example) directly or you can 
kickstart your Play gRPC project on [Lightbend's Tech Hub](https://developer.lightbend.com/start/?group=play&project=play-scala-grpc-example). 

## Configuring your project

Add the Akka gRPC plugin on `project/plugins.sbt` 

```scala
// Akka GRPC
addSbtPlugin("com.lightbend.akka.grpc" %% "sbt-akka-grpc" % "0.4.2")
```

and enable it on your project (in `build.sbt`):

```scala
lazy val `play-scala-grpc-example` = (project in file("."))
  .enablePlugins(PlayScala)
  .enablePlugins(AkkaGrpcPlugin) // enables source generation for gRPC
  .enablePlugins(PlayAkkaHttp2Support) // enables serving HTTP/2 and gRPC
``` 

The `AkkaGrpcPlugin` locates the gRPC `.proto` files and generates source code from it. Remember to enable the plugin 
in all the projects of your build that want to use it.

Note how the `PlayAkkaHttp2Support` is also enabled. gRPC requires HTTP/2 transport and Play supports it only as an opt-in plugin.

## Limitations

As gRPC requires using HTTP/2 only some setups are supported:
 
1. you may use `sbt runProd` to run Play locally in PROD mode, or
1. you may use `./ssl-play run` to run Play in DEV mode.

Note: `ssl-play` is a wrapper around `sbt` enabling extra options on the `sbt` process so Play's Dev Mode 
supports SSL and HTTP/2.

## Running

Running this application requires [sbt](http://www.scala-sbt.org/). gRPC, in turn, requires the transport to be 
HTTP/2 so we want Play to use HTTP/2 (which, in Play, implies HTTPS). These requirements limit which setups are 
supported to run Play and only the following can be used at the moment:

1. you may use `sbt runProd` to run Play locally in a forked JVM in PROD mode, or
1. you may use `./ssl-play run` to run Play in DEV mode within `sbt`.

`ssl-play` is a wrapper script around `sbt` that sets up the ALPN agent (required for HTTP/2) on the JVM running `sbt`.  

In both execution modes above, `sbt` will also generate the server and client sources based on the `app/proto/*.proto` 
files, which happens thanks to the Akka gRPC plugin being enabled. 

Finally, for your convenience, a self-signed certificate is provided in this example (see `conf/selfsigned.keystore`). Setting 
up a keystore works different in DEV mode and PROD mode. Locate the `play.server.https.keyStore.path` setting in 
`application.conf` and `build.sbt` for an example on how to set the keystore on each environment.

## Learning how it works

This example runs a Play application which serves both HTTP/1.1 and gRPC (over HTTP/2) enpoints. This application also
uses an Akka-gRPC client to send a request to itself. When you sent a `GET` request `/` the request is handled by a 
vanilla Play `Controller` that sends a request over gRPC to the gRPC endpoint:


```
                   ---------------
                   |              | 
 -- (HTTP/1.1) --> > Controller  --> --+
                   |              |    |
                   |              |    |
         +-------> > gRPC Router  |    |
         |         |              |    |
         |         ----------------    |
         |                             |
         +------------ (HTTP/2) -------+

```
  

### Serving (Akka) gRPC Services

Have a look at the `conf/routes` file where you'll notice how to embed a gRPC router within a normal play application. 
You can in fact mix normal Play routes with gRPC routers like this to offer a mixed service. You'll notice that we 
bind the `/` path to the `controllers.HomeController` like usual route,
and then we use the `->` router binding syntax to bind the `routers.HelloWorldRouter`. This is because gRPC services 
have paths correspond to their "methods", yet this is handled by its internal infrastructure and end-users need
not concern themselves about the exact names – clients too are generated from the appropriate 
`app/protobuf/helloworld.proto` file after all.

You can read more about [Service gRPC from a Play App](https://developer.lightbend.com/docs/akka-grpc/current/play-framework.html#serving-grpc-from-a-play-framework-app) in the docs.

### Injecting Akka gRPC Clients 

Similarily to the server side, the sources are generated by the Akka gRPC plugin by having it configured to emit the client as well:

```
// build.sbt
akkaGrpcExtraGenerators += PlayScalaClientCodeGenerator,
``` 

In order to make the gRPC clients easily injectable, we need to enable the following module in Play as well (in this 
example app this has been done already though):

```
// application.conf
play.modules {
  # To enable Akka gRPC clients to be @Injected
  enabled += example.myapp.helloworld.routers.AkkaGrpcClientModule
}
```

Which in turn allows us to inject clients to any of the services defined in our `app/proto` directory, just like so:

```scala
import akka.stream.Materializer
import example.myapp.helloworld.grpc.{HelloReply, HelloRequest}
import javax.inject.Inject

import scala.concurrent.Future


class HomeController @Inject() (mat: Materializer, greeterServiceClient: GreeterServiceClient) extends InjectedController {
  implicit val ec = mat.executionContext
}
```

Since you may want to configure what service discovery or hardcoded location to use for each client, you may do so 
as well in `conf/application.conf`, though we will not dive into this here. Refer to the documentation on 
[using Akka Discovery for endpoint discovery](https://developer.lightbend.com/docs/akka-grpc/current/client/configuration.html#using-akka-discovery-for-endpoint-discovery) for more details.

## Verifying

Finally, since now we know what the application is: an HTTP endpoint that hits its own gRPC endpoint to reply to the incoming request. 
We can trigger such request and see it correctly reply with a "Hello Caplin!" (which is the name of a nice Capybara, google it):

```
$ curl --insecure https://localhost:9443 ; echo
Hello Caplin!
```
