---
title: "Using Plugins in Go"
date: 2022-10-27T10:19:29-04:00
---

## Draft...

```
Draft Notes

* Heavily revise notes about go-plugins in general and research/talk a bit more about hashicorp go plugin.
* Clean up template example and explanation
* Link small demo github repo for easy try out
* Follow up and push to get patch merged to go-swagger
```

## The Need

In the last few years I have designed many APIs, primarily relying on [go-swagger](TODO) as a code and documentation generation tool.

Most recently while helping to build a new API Framework at [ScyllaDB](https://www.scylladb.com) I came across the need to perform some string manipulation in the code generation templates that wasn't supported by go-swagger. After playing with a few different approaches to solving this particular use case I settled on adding the very nice [sprig](https://masterminds.github.io/sprig/) library [to go-swagger](https://github.com/go-swagger/go-swagger/pull/2741). This adds a nice set of templating functions to the templating engine of the project and provided exactly what I needed - simply a `trimSuffix` and `trimPrefix` function.

But what if you still need more functionality or have a more unique use case for templating? And what if it is something that doesn't add value as a contribution to the upstream project? This got me thinking about a more modular way to add template functions that doesn't require making a change to the go-swagger codebase each time. With a bit of digging and experimenting, I found go plugins and realized it fit this use case perfectly.

## Go Plugins
Go plugins were introduced in [Go 1.8](https://go.dev/doc/go1.8). Before that, there wasn't anything built into the language for adding additional code at runtime, so a few creative projects were started to fill the gaps. The most notable is Hashicorp's [go-plugin](https://github.com/hashicorp/go-plugin) which has been in use for over 9 years in widely adopted projects such as [Packer](https://www.packer.io) and [Terraform](https://www.terraform.io). It's a very cool solution that uses gRPC on a local network and allows the writing of plugins in any language you see fit. Although a very mature solution this seemed to be a bit overkill for the functionality I was going for. I also always prefer using something out of the Go standard library if the functionality exists, so I decided to dig into the native go-plugins solution and see how it could solve the problem.

The way Go plugins work is pretty simple. You write your application using the plugin package to load symbols from a plugin, you write your plugin containing the appropriate symbols, and at runtime, you tell your application which plugin to use. We will go through these steps with a very simple example: adding a [ROT13](https://en.wikipedia.org/wiki/ROT13) function to a Go template funcmap.

First a quick aside to what a `FuncMap` is in Go templates, it is simply a type alias to a map from strings to interfaces{} (or now with go1.18 `any`):
```go
type FuncMap map[string]any
```

In practice, the values in the map should almost always be functions that accept some number of strings and return a single string.

So back to the plugins - let's say we have a FuncMap in our application and we want to allow users to add their functions to it via a plugin. We will define the function signature for our plugin as `func(template.FuncMap)`, and expect it to be named `AddFuncs`. The intention will be that `AddFuncs` will accept the funcmap and simply add new functions to it (or override existing ones).

In our application this is how we go about loading the plugin:
```go
func main() {
	m := template.FuncMap{}
	p, err := plugin.Open(*pluginPath)
	if err != nil {
		panic(err)
	}

	f, err := p.Lookup("AddFuncs")
	if err != nil {
		panic(err)
	}

	f.(func(template.FuncMap))(m)

	t := template.New("demo")
	t.Funcs(m)

	ut, _ := t.Parse(*text)
	ut.Execute(os.Stdout, nil)
	fmt.Println()
}
```

```go
func main() 
func AddFuncsFromPlugin(pluginPath string) error {
    // p will be a *plugin.Plugin
    p, err := plugin.Open(pluginPath)

    if err != nil {
        return err
    }

    // f is a plugin.Symbol - a pointer to a variable or function
    f, err := p.Lookup("AddFuncs")

    if err != nil {
        return err
    }

    // This will panic if your Symbol does not exactly match the function signature 
    f.(func(template.FuncMap))(t.funcs)
    return nil
}
```


Now we write our plugin. A plugin must always contain exactly one main package. Within the plugin, you can define as many functions and variables as you like.
```go
package main

import "text/template"

func rot13(s string) string {
    // perform the rot13
}

func AddFuncs(m template.FuncMap) {
    m["rot13"] = rot13
}
```

Now all there is to do is build the plugin, and build/run our application telling it where the built plugin resides. Building a plugin is as simple as passing a flag to the go build command, and it outputs a shared object (.so) for you to use wherever the plugin is needed.

```sh
go build -buildmode=plugin -o plugin.so plugin/plugin.go
```

And to see it in action:
```sh
➜ go run main.go -plugin plugin.so -text "{{ rot13 \"Hello, World!\" }}"
Uryyb, Jbeyq!
```

This functionality has now been [merged into go-swagger](https://github.com/go-swagger/go-swagger/pull/2745) and is available to anyone who needs to use custom template processing with their API generation. There are certainly many more great use cases for plugins in Go and I look forward to seeing if the feature gets any more adoption and what else is done with it in the future!