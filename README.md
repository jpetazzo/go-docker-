![QR code for this page](qrcode.png)

# Go + Docker = ♥

Hi! Here are a few interesting and fun things to do with Go and Docker.


## Go without `go`

... And by that, we mean "Go without installing `go`".

Most of you certainly have the Go compiler and toolchain installed,
so you might be wondering "what's the point?"; but there are
a few scenarios where this can still be very useful.

* You still have this old Go 1.2 on your machine (that you can't
  or won't upgrade), and you have to work on this codebase that
  requires a newer version of the toolchain.
* You want to play with cross compilation features of Go 1.5
  (for instance, to make sure that you can create OS X binaries
  from a Linux system).
* You want to have multiple versions of Go side-by-side, but don't
  want to completely litter your system.
* You want to be 100% sure that your project and all its dependencies
  download, build, and run fine on a clean system.

If any of this is relevant to you, then let's call Docker to the rescue!


### Compiling a program in a container

When you have installed Go, you can do `go get -v github.com/user/repo` 
to download, build, and install a library. (The `-v` flag is just
here for verbosity, you can remove it if you prefer your
toolchain to be swift and silent!)

You can also do `go get github.com/user/repo/...` (yes, that's 
three dots) to download, build, and install all the things in 
that repo (including libraries and binaries).

We can do that in a container!

Try this:

```bash
docker run golang go get -v github.com/golang/example/hello/...
```

This will pull the `golang` image (unless you have it already;
then it will start right away), and create a container based on
that image. In that container, `go` will download a little
"hello world" example, build it, and install it. But it will
install it in the container ... So how do we run that program now?


### Running our program in a container

One solution is to *commit* the container that we just built,
i.e. "freeze" it into a new image:

```bash
docker commit $(docker ps -lq) awesomeness
```

Note: `docker ps -lq` outputs the ID (and only the ID!) of
the last container that was executed. If you are the only
uesr on your machine, and if you haven't created another
container since the previous command, that container
should be the one in which we just built the "hello world"
example.

Now, we can run our program in a container based on
the image that we just built:

```bash
docker run awesomeness hello
```

The output should be `Hello, Go examples!`.


#### Bonus points

When creating the image with `docker commit`, you can
use the `--change` flag to specify arbitrary [Dockerfile](
https://docs.docker.com/engine/reference/builder/) commands.
For instance, you could use a `CMD` or `ENTRYPOINT` command
so that `docker run awesomeness` automatically executes
`hello`.


### Running in a throwaway container

What if we don't want to create an extra image just to
run this Go program?

We got you covered:

```bash
docker run --rm golang sh -c \
    "go get github.com/golang/example/hello/... && exec hello"
```

Wait a minute, what are all those bells and whistles?

* `--rm` tells to the Docker CLI to automatically issue a
  `docker rm` command once the container exits. That way,
  we don't leave anything behind ourselves.
* We chain together the build step (`go get`) and the
  execution step (`exec hello`) using the shell logical
  operator `&&`. If you're not a shell aficionado, `&&`
  means "and". It will run the first part `go get...`,
  and if (and only if!) that part is successful, it will run
  the second part (`exec hello`). If you wonder why this
  is like that: it works like a lazy `and` evaluator,
  which needs to evaluate the right hand side
  only if the left hand side evaluates to `true`.
* We pass our commands to `sh -c`, because if we were to
  simply do `docker run golang "go get ... && hello"`,
  Docker would try to execute the program named `go SPACE get
  SPACE etc.` and that wouldn't work. So instead, we start
  a shell and instruct the shell to execute the command
  sequence.
* We use `exec hello` instead of `hello`: this will replace
  the current process (the shell that we started) with the
  `hello` program. This ensures that `hello` will be PID 1
  in the container, instead of having the shell as PID 1
  and `hello` as a child process. This is totally useless
  for this tiny example, but when we will run more useful
  programs, this will allow them to receive external signals
  properly, since external signals are delivered to PID 1 of
  the container. What kind of signal, you might be wondering?
  A good example is `docker stop`, which sends `SIGTERM` to
  PID 1 in the container.


### Installing on our system

OK, so what if we want to run the compiled program on our
system, instead of in a container?

We could copy the compiled binary out of the container.
Note, however, that this will work only if our container
architecture matches our host architecture; in other words,
if we run Docker on Linux. (I'm leaving out people who
might be running Windows Containers!)

The easiest way to get the binary out of the container
is to map the `$GOPATH/bin` directory to a local directory.
In the `golang` container, `$GOPATH` is `/go`. So we can do
the following:

```bash
docker run -v /tmp/bin:/go/bin \
  golang go get github.com/golang/example/hello/...
/tmp/bin/hello
```

If you are on Linux, you should see the `Hello, Go examples!` message.
But if you are, for instance, on a Mac, you will probably see:

```
-bash:
/tmp/test/hello: cannot execute binary file
```

What can we do about it?


### Cross-compilation

Go 1.5 comes with [outstanding out-of-the-box cross-compilation abilities](
http://dave.cheney.net/2015/08/22/cross-compilation-with-go-1-5), so if your
container operating system and/or architecture doesn't match your system's,
it's no problem at all!

To enable cross-compilation, you need to set `GOOS` and/or `GOARCH`.

For instance, assuming that you are on a 64 bits Mac:

```bash
docker run -e GOOS=darwin -e GOARCH=amd64 -v /tmp/crosstest:/go/bin \
  golang go get github.com/golang/example/hello/...
```

The output of cross-compilation is not directly in `$GOPATH/bin`,
but in `$GOPATH/bin/$GOOS_$GOARCH`. In other words, to run the
program, you have to execute `/tmp/crosstest/darwin_amd64/hello`.


### Installing straight to the `$PATH`

If you are on Linux, you can even install directly to your system
`bin` directories:

```bash
docker run -v /usr/local/bin:/go/bin \
  golang get github.com/golang/example/hello/...
```

However, on a Mac, trying to use `/usr` as a volume will not
mount your Mac's filesystem to the container. It will mount
the `/usr` directory of the Moby VM (the small Linux VM
hidden behind the Docker whale icon in your toolbar).

You can, however, use `/tmp` or something in your home
directory, and then copy it from there.


## Building lean images

The Go binaries that we produced with this technique are
*statically linked*. This means that they embed all the code
that they need to run, including all dependencies. This
contrasts with *dynamically linked* programs, which don't
contain some basic libraries (like the "libc") and use a system-wide
copy which is resolved at run time.

This means that we can drop our Go compiled program in
a container, *without anything else*, and it should work.

Let's try this!


### The `scratch` image

There is a special image in the Docker ecosystem: `scratch`.
This is an empty image. It doesn't need to be created or
downloaded, since by definition, it is empty.

Let's create a new, empty directory for our new Go lean image.

In this new directory, create the following Dockerfile:

```dockerfile
FROM scratch
COPY ./hello /hello
ENTRYPOINT ["/hello"]
```

This means:
* start *from scratch* (an empty image),
* add the `hello` file to the root of the image,
* define this `hello` program to be the default thing to execute
  when starting this container.

Then, produce our `hello` binary as follows:

```bash
docker run -v $(pwd):/go/bin --rm \
  golang go get github.com/golang/example/hello/...
```

Note: we don't need to set `GOOS` and `GOARCH` here, because
precisely, we want a binary that will run *in a Docker container*,
not on our host system. So leave those variables alone!

Then, we can build the image:

```bash
docker build -t hello .
```

And test it:

```bash
docker run hello
```

(This should display `Hello, Go examples!`.)

Last but not least, check the image's size:

```bash
docker images hello
```

If we did everything right, this image should be about 2 MB. Not bad!


### Building something without pushing to GitHub

Of course, if we had to push to GitHub each time we wanted to compile,
we would waste a lot of time.

When you want to work on a piece of code and build it within a container,
you can mount a local directory to `/go` in the `golang` container, so that the
`$GOPATH` is persisted across invocations: `docker run -v $HOME/go:/go golang ...`.

But you can also mount local directories to specific paths, to "override" some
packages (the ones that you have edited locally). Here is a complete example:

```bash
# Adapt the two folowing environment variables if you are not running on a Mac
export GOOS=darwin GOARCH=amd64
mkdir go-and-docker-is-love
cd go-and-docker-is-love
git clone git://github.com/golang/example
cat example/hello/hello.go
sed -i .bak s/olleH/eyB/ example/hello/hello.go
docker run --rm \
  -v $(pwd)/example:/go/src/github.com/golang/example \
  -v $(pwd):/go/bin/${GOOS}_${GOARCH} \
  -e GOOS -e GOARCH \
  golang go get github.com/golang/example/hello/...
./hello
# Should display "Bye, Go examples!"
```


## The special case of the `net` package and CGo

Before diving into real-world Go code, we have to confess something:
we lied a little bit about the static binaries. If you are using CGo,
or if you are using the `net` package, the Go linker will generate
a dynamic binary. In the case of the `net` package (which a *lot*
of useful Go programs out there are using indeed!), the main culprit
is the DNS resolver. Most systems out there have a fancy, modular name
resolution system (like the *Name Service Switch*) which relies on
plugins which are, technically, dynamic libraries. By default,
Go will try to use that; and to do so, it will produce dynamic
libraries.

How do we work around that?


### Re-using another distro's libc

One solution is to use a base image that *has* the essential
libraries needed by those Go programs to function. Almost any
"regular" Linux distro based on the GNU libc will do the trick.
So instead of `FROM scratch`, you would use `FROM debian` or
`FROM fedora`, for instance. The resulting image will be much
bigger now; but at least, the bigger bits will be shared with
other images on your system.

Note: you *cannot* use Alpine
in that case, since Alpine is using the musl library instead 
of the GNU libc.


### Bring your own libc

Another solution is to surgically extract the files needed,
and place them in your container with `COPY`. The resulting
container will be small. However, this extraction process
leaves the author with the uneasy impression of a really
dirty job, and they would rather not go into more details.

If you want to see for yourself, look around `ldd` and the
Name Service Switch plugins mentioned earlier.


### Producing static binaries with `netgo`

We can also instruct Go to *not* use the system's libc, and
substitute Go's `netgo` library, which comes with a native
DNS resolver.

To use it, just add `-tags netgo -installsuffix netgo` to
the `go get` options.

- `-tags netgo` instructs the toolchain to use netgo.
* `-installsuffix netgo` will make sure that the resulting
  libraries (if any) are placed in a different, non-default
  directory. This will avoid conflicts between code built
  with and without netgo, if you do multiple `go get`
  (or `go build`) invocations. If you build in containers 
  like we have shown so far, this is not strictly necessary,
  since there will be no other Go code compiled in this
  container, ever; but it's a good idea to get used to it,
  or at least know that this flag exists.


## The special case of SSL certificates

There is one more thing that you have to worry about if
your code has to validate SSL certificates; for instance
if it will connect to external APIs over HTTPS. In that
case, you need to put the root certificates in your
container too, because Go won't bundle those into your
binary.


### Installing the SSL certificates

Three again, there are multiple options available, but
the easiest one is to use a package from an existing
distribution.

Alpine is a good candidate here because it's so tiny.
The following `Dockerfile` will give you a base image
that is small, but has an up-to-date bundle of root
certificates:

```dockerfile
FROM alpine:3.4
RUN apk add --no-cache ca-certificates apache2-utils
```

Check it out; the resulting image is only 6 MB!

Note: the `--no-cache` option tells `apk` (the Alpine
package manager) to get the list of available packages
from Alpine's distribution mirrors, without saving it
to disk. You might have seen Dockerfiles doing something
like `apt-get update && apt-get install ... && rm -rf /var/cache/apt/*`;
this achieves something equivalent (i.e. not leave package
caches in the final image) with a single flag.

*As an added bonus,* putting your application in a container
based on the Alpine image gives you access to a ton of really
useful tools: now you can drop a shell into your container
and poke around while it's running, if you need to!


## Rewriting simple micro-services in Go

Still with us? Great! Now that we know how to build simple Go programs
and put them in lean containers, we propose to take an existing application
and rewrite it in Go to see the insane space savings that we can achieve.

Don't panic, the application will be pretty simple: it is a demo app
part of a Docker orchestration workshop. It is built around a microservices
architecture. One service is in Ruby, two are in Python, one is in node...
But each of them is less than 50 lines of code.


### Discovering the application

To see what the application is like, just clone the code and run it
with compose:

```bash
git clone git://github.com/jpetazzo/orchestration-workshop
cd orchestration-workshop/dockercoins
docker-compose up -d
```

The build will take a few minutes (or a while, if the internet connection
is slow); then, once the application is running, point your browser
to port 8000 of your Docker installation. If you are on Linux, or using
Docker Mac, that will be http://localhost:8000; with some older betas
of Docker Mac, you might have to use http://docker.local:8000/; and
if you are using Docker Machine,
use `docker-machine ip` to see the IP address of your Docker VM..

The app should show a performance graph that will be around 3-4 units
per second.

You can scale the application with `docker-compose scale worker=3`.
Try with higher values. You will see that the performance peaks at 10
hashes per second.


### Analyzing the performance problem

If you want to go the long route, you can check the [complete slides
of the orchestration workshop](http://view.dckr.info:8080/), but if
you want to dive in and start writing Go immediately, check the next
paragraph!


### What's wrong with this application?

The `rng` service is using Python's default WSGI server, which is
mono-process, mono-request, mono-everything. It can only handle
one request at a time, and it sleeps 0.1s before serving each
request, so that's why the performance graph peaks at 10 hashes/s.


### Solving the performance problem

We want to rewrite the `rng` service in Go! If you want to do some
reverse engineering, go ahead, check the repo, find out where
the source code lives, and try to guess what the `rng` service
does and how to reimplement it in Go. If you want better intructions
(or if you can't find out because the code is too obscure), go
to the next paragraph!


### The `rng` service

The `rng` service (Random Number Generator) is a simple web service
that generates random data. It expects requests to `/<somenumber>`
and generates a random binary payload of size `<somenumber>`.
For instance, when the application is running, you can execute
the following command:

```bash
docker run --net dockercoins_default centos curl http://rng/10
```

This will print 10 random bytes on the terminal (binary data,
so this might garble your terminal, and you probably won't see
exactly 10 characters since some of them might be non printable).

We need to reimplement this service in Go!


### A few hints

We recommend that you check the following packages:
- https://golang.org/pkg/math/rand/
- https://golang.org/pkg/net/http/
- https://golang.org/pkg/time/ (if you want to play ball and
  sleep 0.1s before serving each request!)

If you do it "right", you will either see the same performance
(if you handle one request at a time) or a performance gain
(if you handle concurrent requests).


### Building lean containers with Docker Compose

Note that instead of just building with `docker-compose build`, now
you need to build in two steps:
* first, compile your Go binaries;
* then, build your Docker images using those binaries.

## Hacking on Docker itself

If you want to work use your Go skills to improve Docker itself, here are a few
suggestions of GitHub issues targeted for Docker 1.12 that could use your help.
These range from the beginner level to the intermediate level.

- networking/swarm mode: Make gossip timeout configurable: https://github.com/docker/docker/issues/24557
- Proposal: Replace secrets with "join tokens": https://github.com/docker/docker/issues/24430
- `docker swarm join` should take multiple addresses: https://github.com/docker/docker/issues/24085
- Improve cli help for --update-delay flag: https://github.com/docker/docker/issues/24083
- [1.12-rc2] daemon.ID should be replaced with swarm NodeID: https://github.com/docker/docker/issues/24095
- Allow forcing all swarm tasks to reschedule for a service: https://github.com/docker/docker/issues/24469
- docker swarm update without arguments should show usage: https://github.com/docker/docker/issues/24352
- Increase integration test coverage of swarm mode: https://github.com/docker/docker/issues/24240
- Swarm Integration Test Suite is Poorly Documented: https://github.com/docker/docker/issues/24564

Of course, there are plenty of other things to work on. You can look at
Docker's issue tracker on GitHub for ideas: https://github.com/docker/docker/issues

Feel free to ask us for help or advice!
