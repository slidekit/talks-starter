---
##   adapted to github pages (with s6 machinery)
##  see  the original source @ https://github.com/openforce/2017-graphql-meetup 

title:    Introduction to GraphQL with Sangria @ GraphQL Vienna Meetup
author:   Gerhard Hipfinger (@nano4711)
date:     November 13, 2017

## note: change layout to talk - will "automagically" add the slideshow machinery
layout:    talk
---


# Sangria is a Scala GraphQL implementation.

* Based on akka-http
* Seamless integration with Playframework
* Fairly easy implementation model


# What is Scala

* A object orientated and functional hybrid Language
* Statically typed
* Runs in the Java Virtual Machine
* Can use the Java library world


# What is akka-http

A simple functional HTTP implementation.

```scala
object WebServer {
  def main(args: Array[String]) {

    implicit val system = ActorSystem("my-system")
    implicit val materializer = ActorMaterializer()
    implicit val executionContext = system.dispatcher

    val route =
      path("hello") {
        get {
          complete(HttpEntity(ContentTypes.`text/html(UTF-8)`, "<h1>Hello!</h1>"))
        }
      }

    val bindingFuture = Http().bindAndHandle(route, "localhost", 8080)

    println(s"Server online at http://localhost:8080/\nPress RETURN to stop...")
    StdIn.readLine() // let it run until user presses return
    bindingFuture
      .flatMap(_.unbind()) // trigger unbinding from the port
      .onComplete(_ => system.terminate()) // and shutdown when done
  }
}
```


# How to work with Sangria

1. Define a GraphQL schema
2. Test the schema
3. Expose a GraphQL endpoint via HTTP (with akka-http or Playframework)


# Define a GraphQL Schema

```
type Picture {
  width: Int!
  height: Int!
  url: String
}

interface Identifiable {
  id: String!
}

type Product implements Identifiable {
  id: String!
  name: String!
  description: String
  picture(size: Int!): Picture
}

type Query {
  product(id: Int!): Product
  products: [Product]
}
```


# Map it to the Scala world

We use simple case classes

```scala
case class Picture(width: Int, height: Int, url: Option[String])
```

Define the type mapping

```scala
 import sangria.schema._

val PictureType = ObjectType(
  "Picture",
  "The product picture",

  fields[Unit, Picture](
    Field("width", IntType, resolve = _.value.width),
    Field("height", IntType, resolve = _.value.height),
    Field("url", OptionType(StringType),
      description = Some("Picture CDN URL"),
      resolve = _.value.url)))
```

Pretty verbose, uh?


# Scala Macros FTW

Scala supports meta programming with macros (compile time extensions).

```scala
import sangria.macros.derive._

implicit val PictureType =
  deriveObjectType[Unit, Picture](
    ObjectTypeDescription("The product picture"),
    DocumentField("url", "Picture CDN URL"))
```

Sangria comes with nice macros that eliminate the boilerplate.

Usefull for simple cases.


# Now bind akka-http with Sangria

```scala
object Server extends App {
  implicit val system = ActorSystem("sangria-server")
  implicit val materializer = ActorMaterializer()

  import system.dispatcher
  
  val route: Route =
    (post & path("graphql")) {
      entity(as[JsValue]) { requestJson ⇒
        graphQLEndpoint(requestJson)
      }
    } ~
    get {
      getFromResource("graphiql.html")
    }

  Http().bindAndHandle(route, "0.0.0.0", 8080)
}
```

# And now see some action!
