# Scripting Guide

This guide explains the basics of how to create and structure `acid.js` files.

# Acid Scripts, Projects, and Repositories

Acid scripts are stored in an `acid.js` file. They are designed to contain short scripts
that coordinate running multiple related tasks. We like to think of these as
cluster-oriented shell scripts: The script just ties together a bunch of other programs.

At the very core, an Acid script is simply a JavaScript file that is executed within a
special cluster environment. An environment is composed of the following things:

- An Acid server running in your cluster
- A project in which the script will run
- (Optionally) A source code repository (e.g. git) that contains data our
- (Optionally) Other configuration data, like [secrets](secrets.yaml)

For the examples in this document, we have created a project with these values:

vals.yaml
```
cloneURL: https://github.com/deis/empty-testbed.git
namespace: default
project: deis/empty-testbed
repository: github.com/deis/empty-testbed
secrets:
  dbPassword: supersecret
```

And we have installed it into our Acid/Kubernetes cluster with:

```
$ helm install acid/acid-project -f vals.yaml
```

Note that we have linked this project to a GitHub repository. That repository
contains a very simple Node.js application. (Of course, Acid itself does not
care what a repository contains.)

Based on the above, we have:

- A project named `deis/empty-testbed`
- A GitHub repo at `https://github.com/deis/empty-testbed.git`
- And a single secret: `dbPassword: supersecret`

We'll use this project throughout the tutorial as we create some `acid.js` files and test
them out.

## A Basic Acid Script

Here is a basic `acid.js` file that just sends a single
message to the log:

```javascript
console.log("hello world")
```
[acid-01.js](examples/acid-01.js)

If we run this script, we would see output that looked something like this:

```console
Started acid-worker-01brwzs64rve2jvky87hxy1wsp-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
hello world
Done in 1.44s.
```

> Tip: You can use the `lsd` (Lightweight Script Deployer) command to send acid.js
> files to Acid.
>
> In this tutorial we run scripts with `lsd --file acid.js $PROJECT`, where `$PROJECT` is a
> project ID like `acid-830c16d4aaf6f5490937ad719afd8490a5bcbef064d397411043ac`.

In essence, all we have done is started an acid session, logged "hello world", and exited.

This example executes the log in the _global scope_. While it's fine to work in the
global scope, most of the time what we want to do with Acid is write _event handlers_.

## Acid Events and Event Handlers

Acid listens for certain things to happen, such as a new push on a GitHub repository or an
image update on DockerHub. (The events that it listens for are configured in your
project).

When Acid observes such an event, it will load the `acid.js` file and see if there
is an _event handler_ that matches the _event_.

Using the `lsd` tool introduced above, we can see how this works.

The `lsd` tool triggers an `exec` event (short for _execute_) on Acid. So our
`acid.js` file can intercept this event and respond to it:

```javascript
const { events } = require("libacid")

events.on("exec", () => {
  console.log("==> handling an 'exec' event")
})
```
[acid-02.js](examples/acid-02.js)

> The `() => {}` syntax is a newer way to declare an anonymous function. We use this
> syntax throughout the tutorial, but unless otherwise noted, you can substitute in the
> traditional syntax: `function () {}`.

There are a few things to note about this script:

- It imports the `events` object from `libacid`. Almost all Acid scripts do this.
- It declares a single event handler that says "on an 'exec' event, run this function".

Similarly to our first script, this function simply dumps a message to a log.

The above produces something like this:

```console
Started acid-worker-01brx1z21af78hsj4q55anycpc-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx1z21af78hsj4q55anycpc-master
==> handling an 'exec' event
Destroying PVC named acid-worker-01brx1z21af78hsj4q55anycpc-master
Done in 1.49s.
```

One of the strengths of Acid is that we can handle different events in the same file.
Acid will only trigger the appropriate events. For example, we can expand the example
above to also provide a handler for the GitHub `push` event:

```javascript
const { events } = require("libacid")

events.on("exec", () => {
  console.log("==> handling an 'exec' event")
})

events.on("push", () => {
  console.log(" **** I'm a GitHub 'push' handler")
})
```
[acid-03.js](examples/acid-03.js)

Now if we re-run our `lsd` command, we will see the same output as before:

```
Started acid-worker-01brx5m1yppb0dxn4emk76jqtv-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx5m1yppb0dxn4emk76jqtv-master
==> handling an 'exec' event
Destroying PVC named acid-worker-01brx5m1yppb0dxn4emk76jqtv-master
Done in 1.21s.
```

Since Acid did not see a `push` event, it did not execute the `push` event handler.
It only executed the `exec` handler that `lsd` causes.

### Where Do Events Come From?

In order to be able to write good Acid scripts, we need to know what events we
can listen for. So what is the origin of these events?

To answer this question, we need a tiny bit of background knowledge about Acid. Acid has
several components in its runtime. The _worker_ runs the JavaScript that we are writing
here. The _controller_ starts and manages the _worker_. But there are also Acid
_gateways_.

An Acid gateway is a piece of code that executes in your cluster and generates events.
Acid comes with a built-in gateway called `acid-gateway`. This gateway provides the
following:

- Global
  - ping
  - after
  - error
- GitHub hooks:
  - push
  - pull_request
- DockerHub/ACR hooks:
  - imagePush

The `lsd` client declares its own hook (`exec`).

You can create your own gateways which generate their own events, or use gateways created
by others.

In the list above, there are two special events that have very specific usage: `after` and
`error`.

### Two Special Events: _after_ and _error_

The `after` and `error` events are what are called _lifecycle hooks_. They let you
execute some code when Acid hits a certain stage of processing.

The `after` hook runs _at the very end_ of any session that did not die from an error.

```javascript
const { events } = require("libacid")

events.on("exec", () => {
  console.log("==> handling an 'exec' event")
})

events.on("after", () => {
  console.log(" **** AFTER EVENT called")
})
```
[acid-04.js](examples/acid-04.js)

The `lsd` client will trigger the `exec` event. But then when that event has been
processed, Acid will trigger an `after` event before returning:

```console
Started acid-worker-01brx76gx5v3e8r6vbmzcda7q9-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx76gx5v3e8r6vbmzcda7q9-master
==> handling an 'exec' event
 **** AFTER EVENT called
Destroying PVC named acid-worker-01brx76gx5v3e8r6vbmzcda7q9-master
Done in 1.19s.
```

Note that the `**** AFTER EVENT called` message is printed right before the shutdown
begins.

The `error` event is similar, but it is _only_ triggered when a script encounters an
error.

The `after` and `error` events give you an opportunity to do some final processing (like
maybe sending a message to your chat server) before exiting.

At this point, we've taken a look at Acid's event system. But we haven't really done
anything with Acid except log a few messages. Let's turn out attention toward Acid's
ability to "script up" containers.

## Jobs and Containers

A typical Acid script divides up the workload like this:

- An _event handler_ declares how an event (`push`, `exec`) is processed
- When Acid triggers an event, it creates a new `build`, which you can think of as "a
  specific run of your `acid.js` file.
- A build has several _jobs_, where each job starts a container
- A job runs zero or more _tasks_ inside of a container

And in the next section, we'll talk about how we can _group_ jobs.

In the last section, we focused on the event handlers, and we ran several builds that just
logged messages. In this section, we'll create a few jobs.

To start with, let's create a simple job that doesn't really do any work:

```javascript
const { events, Job } = require("libacid")

events.on("exec", () => {
  var job = new Job("do-nothing", "alpine:3.4")

  job.run()
}
```
[acid-05.js](examples/acid-05.js)

The first thing to note is that _we have changed the first line_. In addition to importing
the `events` object, we are now also importing `Job`. `Job` is the prototype for creating
all jobs in Acid.

Next, inside of our `exec` event handler, we create a new `Job`, and we give it two pieces
of information:

- a _name_: The name must be unique to the event handler and should be composed of
  letters, numbers, and dashes (`-`).
- an _image_: This can be _any image that your cluster can find_. In the case above, we
  use the image named `alpine:3.4`, which is fetched [straight from DockerHub](https://hub.docker.com/_/alpine/).

The image is a crucial part of the Acid ecosystem. A container is created from this
image, and it's within this container that we do the "real work". At the beginning, we
explained that we think of Acid scripts as "shell scripts for your cluster." When you
execute a shell script, it is typically some glue code that manages calling one or more
external programs in a specific way.

```
#!/bin/bash

ps -ef "hello" | grep chrome
```

The script above really just organizes the way we call two existing programs (`ps` and
`grep`). Acid does the same, except instead of executing _programs_, it executes
_containers_. Each container is expressed as a Job, which is a wrapper that knows how to
execute containers.

So in our example above, we create a Job named "do-nothing" that runs an Alpine Linux
container and (as the name implies) does nothing.

Jobs are created and run in different steps. This way, we can do some preparation on our
job (as we will see in a moment) before executing it.

To run a job, we use the Job's `run()` method. Behind the scenes, Acid will start a new
`alpine:3.4` pod (downloading it if necessary), execute it, and monitor it. When the
container is done executing, the run is complete.

It is worth noting that a `run()` method is _asynchronous_. That means that when we call
`run()`, it will start processing in the background. Later on, we will see how we can take
advantage of that to create pipelines.

If we run the code above, we'll get output that looks something like this:

```console
Started acid-worker-01brx7v6wsg31k81x0h4pznv47-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx7v6wsg31k81x0h4pznv47-master
looking up default/github-com-deis-empty-testbed-do-nothing
Creating secret do-nothing-1504055331341-master
Creating Job Cache PVC github-com-deis-empty-testbed-do-nothing
Creating pod do-nothing-1504055331341-master
Timeout set at 900000
default/do-nothing-1504055331341-master phase Pending
default/do-nothing-1504055331341-master phase Pending
default/do-nothing-1504055331341-master phase Succeeded
Destroying PVC named acid-worker-01brx7v6wsg31k81x0h4pznv47-master
Done in 5.12s.
```

That's quite a lot of new output for a program that "does nothing". The important part is
this: `Creating pod do-nothing-1504055331341-master`. That tells us that it has taken our
Job and packaged it up as what Kubernetes calls a _pod_. That means it has been scheduled
for execution.

For a few lines we will see messages that let us know that our job isn't running yet, but
is in state `Pending`. Then it will run, and if it runs to a successful completion, it
will be marked as `Succeeded`.

After that, our job is cleaned up, and so is the build.

> Tip: You can see the actual output of each Job through the Acid user interface, or using
> the Kubernetes `kubectl` client: `kubectl logs do-nothing-1504055331341-master`

Basically, our simple build just created an empty Alpine Linux pod which had nothing to do
and so exited immediately.

### Adding Tasks to Jobs

To make our Job do more, we can add tasks to it. A task is an _individual step to be run
inside of the Job's container_.

```javascript
const { events, Job } = require("libacid")

events.on("exec", () => {
  var job = new Job("do-nothing", "alpine:3.4")
  job.tasks = [
    "echo Hello",
    "echo World"
  ]

  job.run()
})
```
[acid-06.js](examples/acid-06.js)

This time we have added some tasks by adding them to the tasks array: `job.tasks = [ /* ... */]`
It will run the `echo` command twice. If we run this new script, its output will look just
about the same as when we ran no tasks:

```console
Started acid-worker-01brx98hq5f3e93jxy5ddpfwgx-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx98hq5f3e93jxy5ddpfwgx-master
looking up default/github-com-deis-empty-testbed-do-nothing
Creating secret do-nothing-1504056818776-master
Creating Job Cache PVC github-com-deis-empty-testbed-do-nothing
Creating pod do-nothing-1504056818776-master
Timeout set at 900000
default/do-nothing-1504056818776-master phase Pending
default/do-nothing-1504056818776-master phase Succeeded
Destroying PVC named acid-worker-01brx98hq5f3e93jxy5ddpfwgx-master
Done in 5.14s.
```

But this time, we can take a look at the logs for our pod and see the results of our
tasks. We will do this using the `kubectl` tool, though you can also use the Acid UI.

```console
$ kubectl logs do-nothing-1504056818776-master
Hello
World
```

Here's what happened: Our `acid.js` script created a new pod named `do-nothing-1504056818776-master`, which
started `alpine:3.4` and then ran the two `echo` tasks. When it completed, Acid let us
know that it `Succeeded` and then it finished up the build.

Now we can take things one more step and create _two jobs_ that each do something.

```javascript
const { events, Job } = require("libacid")

events.on("exec", () => {
  var hello = new Job("hello", "alpine:3.4")
  hello.tasks = [
    "echo Hello",
    "echo World"
  ]

  var goodbye = new Job("goodbye", "alpine:3.4")
  goodbye.tasks = [
    "echo Goodbye",
    "echo World"
  ]

  hello.run()
  goodbye.run()
})
```
[acid-07.js](examples/acid-07.js)

In this example we create two jobs (`hello` and `goodbye`). Each starts an Alpine
container and prints a couple of messages, then exits.

After defining each one, we run them like this:

```javascript
hello.run()
goodbye.run()
```

Now the output of running this command with `lsd` might be a little surprising:

```
Started acid-worker-01brx9n20bsjxeweggtzb7fpka-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brx9n20bsjxeweggtzb7fpka-master
looking up default/github-com-deis-empty-testbed-hello
looking up default/github-com-deis-empty-testbed-goodbye
Creating secret hello-1504057229136-master
Creating secret goodbye-1504057229149-master
Creating Job Cache PVC github-com-deis-empty-testbed-hello
undefined
Creating Job Cache PVC github-com-deis-empty-testbed-goodbye
undefined
Creating pod hello-1504057229136-master
Creating pod goodbye-1504057229149-master
Timeout set at 900000
Timeout set at 900000
default/hello-1504057229136-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/hello-1504057229136-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/hello-1504057229136-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/hello-1504057229136-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/hello-1504057229136-master phase Pending
default/hello-1504057229136-master phase Pending
default/goodbye-1504057229149-master phase Pending
default/hello-1504057229136-master phase Succeeded
default/goodbye-1504057229149-master phase Succeeded
Destroying PVC named acid-worker-01brx9n20bsjxeweggtzb7fpka-master
Done in 15.17s.
```

What is surprising is that if we look at the output above, we see that both jobs seem to
be running at the same time. This is because when we start a job in Acid, it runs
asynchronously. Another way to phrase that is that jobs run _parallel by default_.

Again, if you want to view the output of each job, you can use the `kubectl logs` command
for each pod.

If we want these two pods to run _in order_, with `hello` running to completion before
`goodbye` starts, we can do that by creating what in JavaScript is called a "Promise
chain":

```javascript
const { events, Job } = require("libacid")

events.on("exec", () => {
  var hello = new Job("hello", "alpine:3.4")
  hello.tasks = [
    "echo Hello",
    "echo World"
  ]

  var goodbye = new Job("goodbye", "alpine:3.4")
  goodbye.tasks = [
    "echo Goodbye",
    "echo World"
  ]

  hello.run().then(() => {
    goodbye.run()
  })
})
```
[acid-08.js](examples/acid-08.js)

The important new part is at the end. We have replaced this:

```javascript
hello.run()
goodbye.run()
```

The new version looks like this:

```javascript
hello.run().then(() => {
  goodbye.run()
})
```

And we can read it like this: "run hello, then run goodbye". In the Groups section below,
we will see a simpler way of doing this. But for now, this is one way of running jobs in
sequence.

Now if we run our `acid.js`, the output will look like this:

```console
Started acid-worker-01brxgx62zey1b31ae2ccd2xnm-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brxgx62zey1b31ae2ccd2xnm-master
looking up default/github-com-deis-empty-testbed-hello
Creating secret hello-1504059196321-master
Creating Job Cache PVC github-com-deis-empty-testbed-hello
Creating pod hello-1504059196321-master
Timeout set at 900000
default/hello-1504059196321-master phase Pending
default/hello-1504059196321-master phase Succeeded
looking up default/github-com-deis-empty-testbed-goodbye
Creating secret goodbye-1504059200407-master
Creating Job Cache PVC github-com-deis-empty-testbed-goodbye
Creating pod goodbye-1504059200407-master
Timeout set at 900000
default/goodbye-1504059200407-master phase Pending
default/goodbye-1504059200407-master phase Succeeded
Destroying PVC named acid-worker-01brxgx62zey1b31ae2ccd2xnm-master
Done in 9.17s.
```

Compared to our previous version, we can see the different. It runs the `hello` job first,
and then runs the `goodbye` job.

Before moving on to talk about groups, though, let's do one short example that does
something useful. Our project points to the [empty testbed](https://github.com/deis/empty-testbed)
repository in GitHub. That repository happens to have a small Node.js application, and we
can write a simple set of tasks to build and run that application.

### Running Tasks Against a Git Repository

Earlier we talked about how a project may have an associated Git repository. And when we
created our project, we pointed it to [a
repository](https://github.com/deis/empty-testbed) that contains a simple Node.js
application. In this example, we are going to work directly with that repository.

Here's our new script:

```javascript
const { events, Job } = require("libacid")

events.on("exec", () => {
  var test = new Job("test-app", "node:8")

  test.tasks = [
    "cd /src/hello",
    "yarn install",
    "node index.js"
  ]

  test.run()
})
```

This time around, we are going to run three tasks:

- `cd /src/hello`: Change directories into the place where our source code is.
- `yarn install`: Install the dependencies for our Node.js app. (Yarn is like pip, Maven,
  Glide or CPAN, but for Node.js.)
- `node index.js`: Run the `index.js` file inside of `/src/hello`.

If we run the script, we'll see the usual output, and it will contain a line like this:

```
Creating pod test-app-1504064455281-master
```

If we use `kubectl` to check out the log for our `test-app` container, we'll see something
like this:

```console
$ kubectl logs test-app-1504064455281-master
yarn install v0.27.5
info No lockfile found.
[1/4] Resolving packages...
[2/4] Fetching packages...
[3/4] Linking dependencies...
[4/4] Building fresh packages...
success Saved lockfile.
Done in 0.14s.
hello world
```

That is the output of our three tasks.

But what exactly is this build doing? Why are we doing a `cd /src/hello`? And where did
that directory even come from?

Here's what is happening: Because our project has a Git repository associated with it,
Acid is automatically getting us a copy of that project and attaching that copy to each
`Job` that we run.

When a repository is checked out, it is stored by default in `/src`. So it is as if we
started every job by doing a `git clone https://github.com/deis/empty-testbed.git /src`.
That means we can start our job knowing that we already have access to everything in our
Git project.

So when we `cd` into `/src/hello`, we're changing into the `hello/` directory in the Git
project. And from there on, we are working with the code from the repository.

Being able to associated a Git repository to a project is a convenient way to provide
version-controlled data to our Acid builds. And it makes Acid a great tool for executing
CI pipelines, deployments, packaging tasks, end-to-end tests, and other DevOps tasks.

Automatically mounting a repository is typically a great feature. But every once in a
while it is useful to _disable_ this behavior. To do that, simply add an attribute to your
job:

```javascript
var job = new Job("test", "node:8")
job.useSource = false
```

You can also change the path where the source is stored. The default is `src/`, but it can
be set to another location:

```javascript
var job = new Job("test", "node:8")
job.mountPath = "/mnt/acid/src"
```

## Groups

Earlier we saw an example of creating and running multiple jobs. We saw a simple form
where we could run two jobs in parallel:

```javascript
job1.run()
job2.run()
```

And we saw a slightly more complicated form where we ran one job and then another:

```javascript
job1.run().then( () => job2.run() )
```

These are two ways to work with individual jobs. But sometimes it is desirable to work
with jobs as if they were a group.

For example, we can run two jobs as an ordered group:

```javascript
const { events, Job, Group } = require("libacid")

events.on("exec", () => {
  var hello = new Job("hello", "alpine:3.4", ["echo hello"])
  var goodbye = new Job("goodbye", "alpine:3.4", ["echo goodbye"])

  Group.runEach([hello, goodbye])
})
```
[acid-10.js](examples/acid-10.js)

There are three things to notice in the example above:

1. We now also import `Group` along with `events` and `Job`.
2. Since we are running a simple list of tasks, we declare the task list in the `Job()`
   constructor.
3. We use `Group.runEach()` to run our tasks.

> Tip: Using `Group.runEach()` is often easier to read than creating a Promise chain.

Group has a couple of useful static methods:

- `Group.runEach()` takes a list of tasks and runs them _in sequence_. It does not mark itself
  as complete until every task has been executed.
- `Group.runAll()` takes a list of tasks and runs them all _in parallel_. It does not mark
  itself complete until all of the tasks have finished.

Both of these methods return Promise objects, so they can be chained. For example, here is
an example that runs a `hello` and `goodbye` jobs _in parallel_, then it runs a
`hello-again` job only after both of the others have completed.

```javascript
const { events, Job, Group } = require("libacid")

events.on("exec", () => {
  var hello = new Job("hello", "alpine:3.4", ["echo hello"])
  var goodbye = new Job("goodbye", "alpine:3.4", ["echo goodbye"])
  var helloAgain = new Job("hello-again", "alpine:3.4", ["echo hello again"])

  Group.runAll([hello, goodbye])
    .then( ()=> {
      helloAgain.run()
    })
})
```

In the above case, `hello` and `goodbye` will run at the same time. But `helloAgain` will
not be started until both of the others have finished.

Using groups, you can create sophisticated pipelines.

Sometimes you may want to declare groups ahead of time and then run them, much as you do
with jobs. This is an alternative to using the Group static methods.

```javascript
const { events, Job, Group } = require("libacid")

events.on("exec", () => {
  var hello = new Job("hello", "alpine:3.4", ["echo hello"])
  var goodbye = new Job("goodbye", "alpine:3.4", ["echo goodbye"])

  var helloAgain = new Job("hello-again", "alpine:3.4", ["echo hello again"])
  var goodbyeAgain = new Job("bye-again", "alpine:3.4", ["echo bye again"])


  var first = new Group()
  first.add(hello)
  first.add(goodbye)

  var second = new Group()
  second.add(helloAgain)
  second.add(goodbyeAgain)

  first.runAll().then( () => second.runAll() )
})
```

The above creates two groups, and then later executes them. The order of execution would
be:

- Run both of the jobs in the `first` group.
- Once those two jobs have both completed, run both of the jobs in the `second` group.

This is the way groups can be used to control the order in which groups of jobs are run.

So far we have looked at the acid.js file, the event registry, and jobs and groups. As we
advance our way through Acid, we will next take a look at how `acid.js` scripts can work
with the AcidEvent and Project objects that are sent to every event handler.

## Working with Event and Project Data

Two pieces of information are passed into every event handler: The event that occurred,
and the project.

### The Acid Event

From the event, we can find out what triggered the event, what data was sent with the
event, and (if there is a repository) what repository commit we should be using.

An event looks like this:

```javascriot
var e = {
  buildID: "acid-worker-01brwzs64rve2jvky87hxy1wsp-master",
  type: "lsd",
  provider: "lsd",
  commit: "master",
  payload: ""
}
```

- `buildID` is a unique per-build ID. Every time a new build is triggered, a new ID will
  be generated.
- `type` is the event type. A GitHub Pull Request, for example, would set type to
  `pull_request`.
- `provider` tells what service triggered this event. A GitHub request will set this to
  `github`.
- `commit` is the commit (revision ID) for the VCS. For Git, this can be a SHA, a branch
  name, or a tag.
- `payload` contains any information that the hook sent when triggering the event. For
  example, a GitHub push request generates [a rather large payload](https://developer.github.com/v3/activity/events/types/#pushevent). Payload is an unparsed string.

When one event triggers another event, the triggered event also gets information about the
thing that caused it. This is available on the `event.cause` field. The `after` and
`error` events, for example, will be able to access the `cause` field and see the event
that triggered them.

### The Project

The project gives us information about the repository, the Kubernetes configuration, and
secrets (environment variables or credentials) that are available to us.

```
{
  "id":"acid-830c16d4aaf6f5490937ad719afd8490a5bcbef064d397411043ac",
  "name":"github.com/deis/empty-testbed",
  "kubernetes":{
    "namespace":"default",
    "vcsSidecar":"acidic.azurecr.io/vcs-sidecar:latest"
  },
  "repo":{
    "name":"deis/empty-testbed",
    "cloneURL":"https://github.com/deis/empty-testbed.git"
  },
  "secrets":{
    "dbPassword":"supersecret"
  }
}
```

- `id` is the project ID, as generated by ACID.
- `name` is the project name that is set when you created the project with `helm`.
- `kuberentes` stores Kubernetes-related fields:
  - `namespace` is the namespace in which Acid runs
  - `vcsSidecar` is the container image that Acid uses internally to check out your VCS
    repository.
- `repo` stores information about your VCS repository.
  - `name` is the name of the repo. GitHub projects are named as org/project.
  - `cloneURL` is the URL Acid will use to clone or fetch the repository.
- `secrets` is where you can store environment variables or secretes. These are set using
  `helm` (see the [Secrets Guide](secrets.yaml).

### Using Event and Project Objects

Both the event and the project are passed to every event handler. Until now, we have
ignored them. But we can write a simple script to show some information about the event
and the project:

```javascript
const { events } = require("libacid")

events.on("exec", (e, p) => {
  console.log(">>> event " + e.type + " caused by " + e.provider)
  console.log(">>> project " + p.name + " clones the repo at " + p.repo.cloneURL)
})
```
[acid-13.js](examples/acid13.js)

If we run the above, we'll see output like this:

```console
Started acid-worker-01brz271ma5h06na0bb5j7d2rm-master
yarn start v0.27.5
$ node prestart.js
prestart: src/acid.js written
$ node --no-deprecation ./dist/src/index.js
Creating PVC named acid-worker-01brz271ma5h06na0bb5j7d2rm-master
>>> event exec caused by lsd
>>> project github.com/deis/empty-testbed clones the repo at https://github.com/deis/empty-testbed.git
Destroying PVC named acid-worker-01brz271ma5h06na0bb5j7d2rm-master
Done in 1.04s.
```

Event and project data should be treated with a little extra care. Things like
`secrets` or event `cloneURL` might not be the sorts of information you want accidentally
displayed.

### Passing Project or Event Data to Jobs

Acid is designed to make it easy for you to extract information from the event and project
and sent it into a job. Here are two ways to share information with jobs:

```javascript
const { events, Job } = require("libacid")

events.on("exec", (e, p) => {
  var echo = new Job("echo", "alpine:3.4")
  echo.tasks = [
    "echo Project " + p.name,
    "echo Event $EVENT_NAME"
  ]

  echo.env = {
    "EVENT_NAME": e.type
  }

  echo.run()
})
```

In the above code, we create a job named `echo` and we run two tasks. In the first, we
directly inject the project name (`p.name`) into the task command before the task is run.

In the second case, we use environment variables to pass a name/value pair into the
command, and then it is evaluated at runtime.

`echo.env` is the place to set environment variables for the container. The variable set
to `EVENT_NAME` there is accessible inside the pod as `$EVENT_NAME`.

If we look at the output of the pod, we'll see this:

```
$ kubectl logs echo-1504074306432-master
Project github.com/deis/empty-testbed
Event exec
```

At this point we have seen how we can access information about the project and event. In
the next section we are going to turn to storage. We are going to see how Acid provides
ways for builds and jobs to share storage space.

## Storing Data with Caches and Shared Space

There are many ways that developers can store and retrieve data from Acid. For example,
object storage systems like Azure Object Storate or Amazon S3 or hosted database
providers. Acid developers may choose to use those tools from within jobs.

But Acid comes with a few built-in options that are useful in writing basic Acid builds,
and which don't require modifying your containers. In this part of the guide, we cover two
built-in shared directories.

### Build Storage (Shared Space)

Each build gets its own shared storage. This is a file path that can be accessed by every
job during the build, but which does not survive after the build has completed.

Storage is always mounted to `/mnt/acid/share` on each job's container.

```javascript
const { events, Job, Group } = require("libacid")

events.on("exec", (e, p) => {
  var dest = "/mnt/acid/share/hello.txt"
  var one = new Job("one", "alpine:3.4", ["echo hello > " + dest])
  var two = new Job("two", "alpine:3.4", ["echo world >> " + dest])
  var three = new Job("three", "alpine:3.4", ["cat " + dest])

  Group.runEach([one, two, three])
})
```

In the script above, jobs `one` and `two` should each write a line to the file
`hello.txt`, which is stored in the shared `/mnt/acid/share` directory. Since this
directory is shared among all three jobs, when the third job runs, it should print out the
results of the other two jobs.

So we can check the output of the third job's log:

```console
$ kubectl logs three-1504091079871-master
hello
world
```

That is exactly what we would expect to see.

Importantly, shared storage space is listed to 50 megabytes of storage per build. This can
be overridden in the project configuration.

> Note: Shared storage is dependent on the underlying Kubernetes cluster. Some Kubernetes
> clusters cannot support dynamically provisioned PVCs. If you run into problems with
> this, consult your Kubernetes admin or the Kubernetes storage documentation.

### Job Caches

The shared storage we saw above only persists as long as the build is running. When the
build is complete, the storage is recycled.

Acid provides a second kind of storage that is designed to improve the speed of individual
jobs by giving them access to a cache.

A _job cache_ provides a place for a job to store data that it can access every time it is
run. This can drastically improve performance for things like dependency caching (during
code builds) or indexing.

A job cache is not enabled by default. A job must declare that it needs a cache.

```javascript
const { events, Job } = require("libacid")

events.on("exec", (e) => {
  var job = new Job("cacher", "alpine:3.4")
  job.cache.enabled = true

  job.tasks = [
    "echo " + e.buildID + " >> /mnt/acid/cache/jobs.txt",
    "cat /mnt/acid/cache/jobs.txt"
  ]

  job.run()
})
```
[acid-16.js](examples/acid-16.js)

This script creates a new job and then enables the cache. Then it runs two different
tasks. The first writes the build's unique ID into a file in the cache, and the second one
echos the contents of that generated file.

If we run the above a few times and then check the job's output, we'll see one ID for each
time the job was run.

```console
$ kubectl logs cacher-1504091963651-master
acid-worker-01brzvq79mj4849vy7ae0fez44-master
acid-worker-01brzvqnhz21yjjh8m0zyxrxsc-master
acid-worker-01brzvrhwe50xtktf4gcqk94wj-master
```

This happens because each time the job runs, the new build ID is written into the same
file that the job used on other builds.

By default, job caches are limited to only 5 megabytes (`5Mi`). However, you can easily
change this by setting `job.cache.size` to a larger value (`50Mi`, `5Gi`).

There are two final observations to make about job caches:

1. While a job cache is designed to persist across multiple runs, they are still
   considered to be _volatile storage_, which means a cache can reset. Do not use it as if
   it were stable long-term storage.
2. Caches are not automatically destroyed by Acid (though other systems may clean them).
   That means that if you add lots and lots of jobs with caches enabled, lots of storage
   space will be reserved even if it is unused.

## Conclusion

This guide covers the basics of working with `acid.js` files. If you are not sure how to
get started with Acid now, you might want to take a look at the [tutorial](../intro). If
you want more details about the Acid JS API, you can look at the [API
documentation](javascript.md).

Happy Scripting!