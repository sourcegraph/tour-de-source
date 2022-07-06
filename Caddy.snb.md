# How Caddy 2 works, a deep dive into the source

![Caddy logo](https://storage.googleapis.com/sourcegraph-assets/blog/caddy-social.jpg)

Caddy is a production web server that prioritizes developer experience and extensibility. One of the magical things that works out of the box is automatic HTTPS. Its modular architecture makes it easy to extend and customize, and it has awesome docs coupled with a simple and straightforward configuration. Caddy is used in a huge range of environments, from small startups to large institutions like hospitals. With over 40,000 stars on GitHub, Caddy is one of the most popular production web servers today.

Key points of interest
* [Caddy's highly modular architecture](https://sourcegraph.com/notebooks/Tm90ZWJvb2s6MTA3NQ==#overview-and-file-layout-446f80ff-8ceb-4a59-b7a6-58c53a4d9b73)
* [The patterns Caddy uses for defining a featureful CLI](https://sourcegraph.com/notebooks/Tm90ZWJvb2s6MTA3NQ==#cli-4e193cb1-7929-4d89-a2a6-8223d5ad6dec)
* [The curious fact that Caddy core knows nothing about HTTP or web protocols](https://sourcegraph.com/notebooks/Tm90ZWJvb2s6MTA3NQ==#ah-configuration-ebc69e78-c0f6-475c-8036-236eb1acb54f)
* [How Caddy handles the magic of automatic HTTPS](https://sourcegraph.com/notebooks/Tm90ZWJvb2s6MTA3NQ==#automatic-https-792ac644-2030-4251-af78-ed0e7e57e7c8)

Let's take a deep dive into how Caddy works and some of the design decisions and patterns that make it a reliable, extensible, and delightful web server.

## Overview

Caddy is distributed as a single static binary. Its command-line interface is defined in the top-level [`cmd`](https://sourcegraph.com/github.com/caddyserver/caddy/-/tree/cmd) directory. Its main function quickly delegates to the `Main` function defined in an importable package, so that Caddy can also be imported and run as a library, not just a standalone executable:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/cmd/caddy/main.go?L36-38

There are two other top-level directories to take note of:

* `caddyconfig` handles the loading and unloading of configuration, which comprises the bulk of the core behavior of Caddy—more on this shortly.
* `modules` contains the definition of all the first-party modules that plug into Caddy and contribute basically all of the functionality you'd expect from a web server, like HTTP handling, TLS, integrations with various DNS providers, load balancing, and SSH.

First-party modules are incorporated into Caddy using the anonymous import pattern:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules/standard/imports.go

## Ah, configuration

One of the interesting things about the design of Caddy is that *its core is just configuration management*. It pushes out all other functionality into modules, so that basically the only thing left in core is accepting and reacting to configuration changes.

The docs have a [nice interactive view of the configuration format](https://caddyserver.com/docs/json/), which uses JSON.

There is a [hierarchical namespace of modules](https://caddyserver.com/docs/extending-caddy/namespaces). Top-level modules are called ["apps"](https://caddyserver.com/docs/json/apps/). For example, the HTTP app has submodules under the `http.handlers` namespace that provide different types of HTTP handlers, which include `authz`, `cache`, and `git`.

Central to this modular architecture are interfaces, which interestingly don't contain much in the way of web-specific functionality. For example, here is the `App` interface:

https://sourcegraph.com/github.com/caddyserver/caddy/-/blob/caddy.go?L85-89

Apps can optionally implement other interfaces, which define more lifecycle methods like `Provision` and `Validate`:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules.go?L294-296

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules.go?L303-305

But all that is *required* from apps is to know how to start and stop themselves. On startup, the initial config is posted to Caddy, which then invokes `Start` on each of the apps defined in the config.

Let's dive more into the source to see how that works.

## CLI

The Caddy CLI interface is defined in `commands.go`. It doesn't use any of the popular third-party CLI declaration libraries, just a simple registration pattern on top of the standard library. Here's where the `caddy run` command is defined and registered:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/cmd/commands.go?L106-154

This passes control to the `cmdRun` function:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/cmd/commandfuncs.go?L155-165

The crux of this function is the `caddy.Load` invocation, which loads the initial config:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/cmd/commandfuncs.go?L212-217

Caddy will then listen on a socket for any config updates. Interestingly, it does this through a module rather than "in core":

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/caddyconfig/load.go?L30-44

This module adds a handler that will listen on the admin port and accept POST requests to the `/load` URL path that contain new configuration. It will then deserialize the new config and pass it to `caddy.Load`:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/caddyconfig/load.go?L54-123

Let's take a look at this `caddy.Load` function. Its job is to accept a new configuration and apply it safely and atomically, spinning up and down modules to reflect the new config state:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/caddy.go?L100-119

The configuration format is defined in the following struct, which reflects the fields in the [JSON config](https://caddyserver.com/docs/json/):

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/caddy.go?L63-83

`caddy.Load` invokes `changeConfig`, which handles spinning up new apps, atomically switching over, and cleaning up old apps. This in turn calls [`unsyncedDecodeAndRun`](https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/caddy.go?L256:6&subtree=true) which eventually makes it down to the invocation of the `Start` method of each Caddy app:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/caddy.go?L447

Here, the switchover happens between new and old config:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/caddy.go?L287-289

A mutex guarantees the switch is atomic:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/caddy.go?L138-139

At its core, Caddy just handles configuration updates. All the "actual" functionality of the webserver is implemented in modules, whose lifecycles begin and end with the invocation of `Start` and `Stop`. Let's look at those next.


## Apps

Caddy modules are namespaced hierarchically. The top-level modules are called apps, and these provide all the actual web server functionality, like handling HTTP, TLS, and other web-related protocols. Again, Caddy core is just configuration management and, aside from the simple handler responsible for accepting and updating configuration, is agnostic of any web protocols.

Each app must provide only a `Start` and `Stop` method. Let's take a look at the app for handling HTTP requests. Here is its `Start` method, which creates each HTTP server:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules/caddyhttp/app.go?L291-309

And here is the `Stop` method, which invokes `Shutdown` on each server:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules/caddyhttp/app.go?L412-436

## Automatic HTTPS

One of the magical features of Caddy is its ability to automatically provision and use certificates for HTTPS. This is a feature of the HTTP app.

HTTP and TLS are distinct protocols, and there is a separate Caddy app to handle TLS-related matters. The HTTP app invokes the TLS app. It does so in a lifecycle method called `Provision`, which apps can optionally define in addition to `Start` and `Stop`. The HTTP app's `Provision` method grabs a reference to the TLS app and stores it in the `tlsApp` field on the `App` struct instance:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules/caddyhttp/app.go?L139-147

The TLS app provides all the TLS services that other Caddy apps may need. [The interactive config](https://caddyserver.com/docs/json/apps/tls/) provides a good overview of its functionality.

<!-- The `certificates` key tells the TLS app how to load certificates and is modular. You can load from files or raw strings, but there's also an `automate` option. -->


The automatic HTTPS setup proceeds in 2 phases. `automaticHTTPSPhase1` handles initial setup, finding domain names in the config and their corresponding certificates:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/modules/caddyhttp/app.go?L154

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/modules/caddyhttp/autohttps.go?L85

It invokes the TLS app using the previously acquired reference to find all matching certificates for each domain name:

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/modules/caddyhttp/autohttps.go?L190

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99b/-/blob/modules/caddytls/tls.go?L426-430

`automaticHTTPSPhase2` completes thet setup, beginning active certificate management for all domain names after the servers have been started.

https://sourcegraph.com/github.com/caddyserver/caddy@4b4e99bdb2e327d553a5f773f827f624181714af/-/blob/modules/caddyhttp/app.go?L402-407

<!-- Loading the http handler module: `/modules/caddyhttp/routes.go`: `RouteList.ProvisionHandlers` : `ctx.LoadModule` -->
<!-- * Uses reflection -->

## Comment on abstraction

It's interesting how generic core interfaces like `App` are in Caddy. There's nothing web-specific about any of them, and you could almost imagine the same interfaces being used for a general framework for any long-running program that must respond to configuration changes. Indeed, the [Caddy website](https://caddyserver.com/docs/getting-started), which auto-generates the docs from source and handles user identity and account management, is entirely implemented as a third-party Caddy module. Several large companies have implemented web services or applications as Caddy modules.

One of Caddy v2's goals was to use an architecture that enabled the addition of many new features that were becoming difficult to add to the growing Caddy v1 codebase. Modularity was the solution to the problem of enabling the long list of features covering the vast array of web protocols. Some might ask if the abstractions in Caddy v2 are *too* generic. In our conversation, Matt Holt, creator of Caddy, noted that he asked himself the same question. "Am I reinventing a sort of userspace operating system? Maybe, but it's really handy." These abstractions were the product of many cycles of iteration and the lessons learned from Caddy v1.

Arguably, the path that Caddy v1 took was the right one for the first version of the system. It was a brand new web server and the abstraction boundaries would have been unclear until the first version was built and validated with actual users.

Caddy v2 serves as a fantastic case study in the modularization that takes place as a successful software system transitions from "version 1" to "version 2", moving from a "let's do one thing well" mentality to a philosophy of enabling effective extensibility.

<!-- TODO Meme: Futurama: "is this a web server or web framework?" -->

## Next steps

* You can get started with Caddy by visiting [caddyserver.com](https://caddyserver.com/).
* Consider sponsoring Caddy's development as an open source project by [contributing to Caddy's creator, Matt Holt](https://github.com/sponsors/mholt).
