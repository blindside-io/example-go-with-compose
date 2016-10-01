# Writing a Go app with Docker Compose

Writing Go applications in an isolated environment with Docker comes with some great advantages. You get a clean `GOPATH`, the bare essentials for developing, and you can easily change which Go version you're developing against.

In this quick tutorial, we're going to show you how to structure a Go application with Docker Compose as your development environment.

## Getting Started

First off, if you haven't installed Docker or gone through some of the basics. You can checkout the free guide on [blindside.io](https://www.blindside.io). It will take you through installation and basics.

## Project Structure

When developing with Docker Compose + Go, it really doesn't matter where you create your project in your machine. If your GOPATH is set to `~/development/go`, you can still create your projects inside of `~/Desktop` and it won't _really_ matter. However, please don't create projects on your desktop.

With that said, I still recommend putting all of your apps in your GOPATH as it makes things consistent and if someone hops on your machine to assist in debugging things, everything should be where they normally are.

Let's create a new project called "go-with-compose" in our GOPATH:

```
$ mkdir $GOPATH/src/go-with-compose
```

Inside of this folder we're going to have a couple of files:

```
docker-compose.yml
main.go
```

To get started, we'll just make a super simple `main.go` with hello world inside of it:

```
# main.go
package main

import "fmt"

func main() {
	fmt.Println("Hello World")
}
```

In our `docker-compose.yml` file, we're going to add a service that uses a Go image like so:

```
version: "2"

services:
	app:
		image: golang:1.6.1-alpine
```

I like using the Alpine variant of the Go image because it's slim but still has everything I normally would use in a Go application.

At this point we can actually run a simple command using compose:

```
$ docker-compose run app go env

GOARCH="amd64"
GOBIN=""
GOEXE=""
GOHOSTARCH="amd64"
GOHOSTOS="linux"
GOOS="linux"
GOPATH="/go"
GORACE=""
GOROOT="/usr/local/go"
GOTOOLDIR="/usr/local/go/pkg/tool/linux_amd64"
GO15VENDOREXPERIMENT="1"
CC="gcc"
GOGCCFLAGS="-fPIC -m64 -pthread -fmessage-length=0"
CXX="g++"
CGO_ENABLED="1"
```

This will print our environment for inside of our container. What we're interested in from this output is the GOPATH for the Go Docker image. Which we see is `/go/src`

Now what we can do is mount our projects directory inside of the container by modifying our `docker-compose.yml` file.

```
version: "2"

services:
  app:
    image: golang:1.6.1-alpine
    volumes:
      - .:/go/src/go-with-compose
```

You can see here that we're mounting the current directory to the directory `/go/src/go-with-compose` inside of the running app container.

Now, what we can do, is change the working directory to the path of our application when we run any command inside of this container:

```
version: "2"

services:
  app:
    image: golang:1.6.1-alpine
    volumes:
      - .:/go/src/go-with-compose
    working_dir: /go/src/go-with-compose
```

This allows us to run Go commands easily for our project like so:

```
$ docker-compose run app go run main.go
```

Huzzah! We've ran our main.go file inside of a container initialized by Docker Compose. But why stop there?

One of the main reasons for docker compose is to easily start your containers with `docker-compose up`. But if we run that no we'll actually see an error as the Go image doesn't define a CMD instruction, and our compose file omits a default command as well.

So what we can do is add a default command to run when starting our application with `up`:

```
version: "2"

services:
  app:
    image: golang:1.6.1-alpine
    volumes:
      - .:/go/src/go-with-compose
    working_dir: /go/src/go-with-compose
    command: go run main.go
```

Now we can run a simple:

```
$ docker-compose up
```

And voila! We see our simple hello world application running.

## Running Tests

With all of the test you've been writing for your Go application (right?!), you'll want to probably run them inside of this environment as well. This is easy to do as well. We can the following command in our terminal:

```
$ docker-compose run app go test -v ./...
```

This should run all of the tests in the directory of your project. `go test` will be ran from the correct directory because of the `working_dir` configuration we provided in our compose yaml.

## Using a Database

A lot of the time we're writing and reading data from a database. In a local setting we'd normally just run the database on our machine as a normal process. However, in Docker land we won't have access to that database. We'll need to start a database in a separate container and connect to that. Luckily this is drop-dead simple with Docker Compose.

Let's add a bit more to our `docker-compose.yml` file:

```
version: "2"

services:
  app:
    image: golang:1.6.1-alpine
    volumes:
      - .:/go/src/go-with-compose
    working_dir: /go/src/go-with-compose
    command: go run main.go
    links:
      - redis

  redis:
    image: redis:alpine
```

We've done 2 things here:

1. Added a `links` declaration to our `app` service definition.
2. Added a new service called `redis` using the image redis alpine.

No we can modify our `main.go` file to connect to Redis and do a simple PING command and print the result:

```
package main

import (
	"fmt"

	redis "gopkg.in/redis.v4"
)

func main() {
	client := redis.NewClient(&redis.Options{
		Addr:     "redis:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})

	pong, err := client.Ping().Result()
	fmt.Println(pong, err)
}
```

Now lets try our our new `docker-compose.yml` file and Go program:

```
$ docker-compose up
```

You'll see the Redis server start up and... an error? Crap.

```
app_1    | main.go:6:2: cannot find package "gopkg.in/redis.v4" in any of:
app_1    | 	/usr/local/go/src/gopkg.in/redis.v4 (from $GOROOT)
app_1    | 	/go/src/gopkg.in/redis.v4 (from $GOPATH)
```

Since our container has a separate file system than our host machine, we need to make sure the redis package is available in the container at runtime. This is easily solved using the vendoring capability as of Go 1.5 (with the vendor experiment feature enabled). Since our Go program is running inside of a 1.6.x Go container, we can simply add the redis package to the `vendor` directory since we're mounting the entire directory into the container at runtime.

## Vendoring our Dependencies

To vendor our projects dependencies, I highly recommend the `govendor` tool (https://github.com/kardianos/govendor).

```
$ go get -u github.com/kardianos/govendor
# Make sure you're in the correct project directory
$ govendor init
$ govendor add +external
```

Now our project should have a `vendor` directory with our dependencies. We should be able to re-try our `docker-compose up`:

```
$ docker-compose up
...
app_1    | PONG <nil>
...
```

If we look closely into the output from our command, we should see the `pong` value returned from the redis server.

## Using an environment variable

Now in our connection configuration, we've declared the address for the Redis server like so:

```
client := redis.NewClient(&redis.Options{
	Addr:     "redis:6379",
	Password: "", // no password set
	DB:       0,  // use default DB
})
```

Not exactly configurable... let's solve this by using environment variables provided by our `docker-compose.yml` file.

In our `docker-compose.yml` file lets add another setting to our `app` service:

```
version: "2"

services:
  app:
    image: golang:1.6.1-alpine
    volumes:
      - .:/go/src/go-with-compose
    working_dir: /go/src/go-with-compose
    command: go run main.go
    links:
      - redis
    environment:
      REDIS_URL: redis:6379

  redis:
    image: redis:alpine
```

Notice how our service name for redis matches up with our hostname for redis. If we declared our redis service as "chicken", our environment variable would be `REDIS_URL: chicken:6379`.

Now in our Go program we'll modify it to use this environment variable to give us a more 12-factor application:

```
client := redis.NewClient(&redis.Options{
	Addr:     os.Getenv("REDIS_URL"),
	Password: "", // no password set
	DB:       0,  // use default DB
})
```

Let's start our application but only with the `app` service logs  displayed. Because our app service explicitly links to the redis service, it will also be started if needed, but with its logs supressed.

```
$ docker-compose up app
```

And we should see

```
docker-compose up app
Starting gowithcompose_redis_1
Starting gowithcompose_app_1
Attaching to gowithcompose_app_1
app_1    | PONG <nil>
gowithcompose_app_1 exited with code 0
```

Great!

## Closing

I hope this tutorial shed some light on how to develop a Go program with Docker + Compose. I've found it incredibly helpful when building services that need to communicate with eachother and the ability to blow away an environment when things get a little wonky and re-build is amazing.

If you want the source code for this project, you can find it here: https://github.com/blindside-io/example-go-with-compose
