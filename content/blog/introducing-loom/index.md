---
date: "2026-02-11T10:24:52+01:00"
title: "INTRODUCING LOOM"
description: "The component framework for Go."
---

After four months in the making, I'm really excited to announce loom.
<br/>
The first signal-based component framework in Go, that can be used for the Web, the Terminal, and more.

```go {style=tokyonight-moon}
func Counter() Node {
	count, setCount := Signal(0)

    go func() {
        for {
            time.Sleep(time.Second / 30)
            setCount(count() + 1)
        }
    }()

	return P(Text("Count: "), BindText(count))
}
```

<br/>

### What is loom?

Loom is a component framework.<br/>
It's similar to modern versions of SolidJS or SvelteJS, but in Go and with a few twists:

<br/>

**1\) Markup is just Go functions.**

Markup is not written in HTML, using templating, or in a separate JSX-like syntax that would require extra tooling.
Instead, it's just plain Go.

A component is simply a function that returns a `loom.Node`, optionally taking children as arguments.<br/>

```go {style=tokyonight-moon}
func MyComponent(children ...loom.Node) loom.Node {
    // ...
}
```

Since markup is _just_ Go functions, it can be used and written however it fits you best to construct a complete UI.

For instance creating a `Card()` component with a title and a body:

```go {style=tokyonight-moon}
func Card(title string, body loom.Node) loom.Node {
    return Div(
        H2(Text(title)),
        body,
    )
}

func App() loom.Node {
    return Div(
        Article(Card("A blog post", Text("A description."))),
        Article(Card("Another blog post", Text("Another description."))),
    )
}
```

<br/>

**2\) Concurrency is a first class citizen.**

Signal-based reactive models have been relying on global state and predictible tasks scheduling to capture signal reads. Making it impossible to work with signals across multiple threads or concurrent code execution.

Loom's [reactive model](https://github.com/loom-go/loom/tree/main/signals#signals) solves this issue while still keeping
the [consistency and reliablilty](https://github.com/AnatoleLucet/sig?tab=readme-ov-file#features) of modern signal-based reactive models.<br/>
With this model you can update signals from hundreds of concurrent tasks, or create and destroy effects/memos across any number of goroutines and threads without any risk of polution.

```go {style=tokyonight-moon}
get, set := Signal(0)

Effect(func() {
    fmt.Println("Changed:", get())
})

go func() {
    // triggers the effect as it should.
    // no risk of another signal sneaking in and poluting the effect.
    set(10)
}()
```

<br/>

**3\) Its not tied to any plateform.**

Loom is not tied to the web, nor to the terminal, nor native.<br/>
By itself, loom only provides the reactive model and basic components like [`For()`](/docs/components/for) and [`Fragment()`](/docs/components/fragment).

The rest is handled by plateform-specific [renderers](/docs/get-started/concepts#renderer) which comes with components and tools for _that_ plateform.

Loom has two official renderers:<br/>
[\*] <a href="/term/intro">LOOM-TERM -></a> | For building Terminal UIs.<br/>
[\*] <a href="/web/intro">LOOM-WEB -></a> | For building Web SPAs.

{{< tabs items="TERM,WEB" >}}
{{< tab >}}

```go {style=tokyonight-moon}
import (
    "github.com/loom-go/loom"
    "github.com/loom-go/term"

    // importing every components from the term renderer
    . "github.com/loom-go/term/components"
)

func App() loom.Node {
    // using the Box() and Text() components from the term renderer
    return Box(Text("Hello World!"))
}

func main() {
    app := term.NewApp()
    app.Run(term.RenderFullscreen, App)
}
```

{{< /tab >}}
{{< tab >}}

```go {style=tokyonight-moon}
import (
    "github.com/loom-go/loom"
    "github.com/loom-go/web"

    // importing every components from the web renderer
    . "github.com/loom-go/web/components"
)

func App() loom.Node {
    // using the P() and Text() components from the web renderer
    return P(Text("Hello World!"))
}

func main() {
    app := web.NewApp()
    app.Run("#root", App)
}
```

{{< /tab >}}
{{< /tabs >}}

<br/>

**4\) Reactivity is explicit.**

Most modern reactive framework have implicit reactivity.<br/>
When you use a signal in your markup, changes to that signal are automatically reflected to the proper element.

Loom takes a different approach. Reactive changes to the tree must be explicitly defined by the user.<br/>

It might seem extra tedious at first, but I promise it's not.<br/>
Explicit reactivity (or [binding](/docs/guides/binding)) gives you more control over the tree and how it reacts to changes.

```go {style=tokyonight-moon}
count, setCount := Signal(0)

return P(
    Text("Reactive count: "),
    BindText(count), // text updates each time the signal changes

    Text("Unreactive count: "),
    Text(count()), // text does not update and only shows initial value
)
```

<br/>

### Showcase

<br/>

<details>
<summary>counter</summary>

{{< tabs items="TERM,WEB" >}}

{{< tab >}}

A very simple counter built with **LOOM-TERM**:

<video src="/medias/counter-term.mp4" autoplay loop muted playsinline></video>

```go {style=tokyonight-moon}
// define your component
func Counter() Node {
	count, setCount := Signal(0)

    // start a new goroutine when the component is mounted
    go func() {
        for {
            time.Sleep(time.Second / 30)
            setCount(count() + 1) // update the signal inside the goroutine
        }
    }()

    // return some markup
	return P(Text("Count: "), BindText(count))
}

func main() {
	app := term.NewApp()

    // render the component using the terminal renderer
    errs := app.Run(term.RenderInline, Counter)
}
```

{{< /tab >}}
{{< tab >}}

A very simple counter built with **LOOM-WEB**:

<video src="/medias/counter-web.mp4" autoplay loop muted playsinline></video>

```go {style=tokyonight-moon}
// define your component
func Counter() Node {
	count, setCount := Signal(0) // define a signal with a getter and setter

    // start a new goroutine when the component is mounted
    go func() {
        for {
            time.Sleep(time.Second / 30)
            setCount(count() + 1) // update the signal inside the goroutine
        }
    }()

    // returns some markup
    // (here using the web renderer's P() and Text() components)
	return P(
        Text("Count: "),
        BindText(count), // render the count signal as text. reactivity is explicit
    )
}

func main() {
	app := web.NewApp()

    // render the component using the web renderer
    errs := app.Run("#root", Counter)
}
```

{{< /tab >}}
{{< /tabs >}}

<br/>
<details>
<summary><i>With goroutine cancellation</i></summary>

```go {style=tokyonight-moon}
func Counter() Node {
	count, setCount := Signal(0)

    go func(self Component) {
        // stop the loop when the component is diposed
        for !self.IsDisposed() {
            time.Sleep(time.Second / 30)
            setCount(count() + 1)
        }
    }(Self())

	return P(Text("Count: "), BindText(count))
}
```

See [`Self()`](/docs/reactivity/self).

</details>
</details>

<b>\-\-\-</b><br/><br/>

<details>
<summary>todolist</summary>

A todolist app built with **LOOM-TERM**:

<video src="todolist.mp4" autoplay loop muted playsinline></video>

Full source code at [term/examples/todolist](https://github.com/loom-go/term/tree/main/examples/todolist).

</details>

<b>\-\-\-</b><br/><br/>

<details>
<summary>console</summary>

An example of using the built-in [`Console()`](/term/components/console) component from **LOOM-TERM**:

<video src="console.mp4" autoplay loop muted playsinline></video>

Full source code at [term/examples/console](https://github.com/loom-go/term/tree/main/examples/console).

</details>

<b>\-\-\-</b><br/><br/>

<details>
<summary>styling</summary>

An example of reactive styling and hover state with **LOOM-TERM**:

<video src="styling.mp4" autoplay loop muted playsinline></video>

Full source code at [term/examples/styling](https://github.com/loom-go/term/tree/main/examples/styling).

</details>

<br/>

### What next?

loom is still in very _very_ early development.<br/>
Expect bugs, but also expect more stability and more features to come!

This initial releases sets the stage for what's to come.<br/>
It proves the idea works and is actually worth pursuing.

The coming weeks/months of development are going to be targeted towards higher stability of the framework,
and better documentation to make loom more accessible for a broader audience.

<br/>

The next core features you should expect to land are going to gravitate around reactive asynchronicity --
a very important step towards proper: fetching, computations, or anything blocking.<br/>
Async tasks are completely possible in the current version of loom, but they could be _better_.

One of the first building blocks in that direction is **async memos**.

Put simply, an async memo is just a reactive computation that's executed in a goroutine.

```go {style=tokyonight-moon}
userID, setUserID := Signal(0)

user := AsyncMemo(func() (User, error) {
    // runs in a goroutine each time the userID signal changes
    return getUser(userID())
})

// use it like a regular signal, but with a possible error
u, err := user()
```

Async memos will open the door for more advanced features like suspense, panic boundaries, and async stores similar to [TanStack Query](https://tanstack.com/query/latest).

<br/>

If you'd like to help, head over at [github.com/loom-go](https://github.com/loom-go/)!

---

<details>
<summary>CREDITS</summary>

The fundamentals behind loom's reactive model rest on the hard work done by [Ryan Carniato](https://github.com/ryansolid), [Milo Mighdoll](https://github.com/milomg) and others pushing the limits of signal-based reactive systems.<br/>
Loom's reactive model started off with a heavily inspired version of what's been done for the upcoming [SolidJS V2 model](https://github.com/solidjs/signals).

The **LOOM-TERM** renderer also rests on the hard work done by [Kommander](https://github.com/kommander) and the OpenCode team on their very cool [native renderer](https://github.com/anomalyco/opentui/tree/24cc57e7c7023e8be2809fee9bd76149aa8d41a7/packages/core/src/zig).
Many of the ideas behind **LOOM-TERM** where inspired by what's been done on OpenTUI.

---

</details>

If you made it here you're probably intrested in how to get started with loom! Head over to the -> [DOCS](/docs) to learn more.

If you have a question or want to discuss something about loom:
come and [join the Discord](https://discord.gg/DfDCFCHp92). There's no such thing as a bad question!
