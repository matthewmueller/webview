# webview

[![Build Status](https://travis-ci.org/zserge/webview.svg?branch=master)](https://travis-ci.org/zserge/webview)
[![Build status](https://ci.appveyor.com/api/projects/status/ksii33qx18d94h6v?svg=true)](https://ci.appveyor.com/project/zserge/webview)
[![GoDoc](https://godoc.org/github.com/zserge/webview?status.svg)](https://godoc.org/github.com/zserge/webview)
[![Go Report Card](https://goreportcard.com/badge/github.com/zserge/webview)](https://goreportcard.com/report/github.com/zserge/webview)


A tiny cross-platform webview library for C/C++/Golang for building modern cross-platform GUI.

It supports two-way JavaScript bindings (to call JavaScript from C/C++/Go and to call C/C++/Go from JavaScript).

It uses Cocoa/WebKit on macOS, gtk-webkit on Linux and good old MSHTML on Windows.

<p align="center"><img alt="linux" src="examples/todo-go/screenshots/screenshots.png"></p>

## Webview for Go developers

If you are interested in writing Webview apps in C/C++ - skip to the next section.

### Getting started

Install Webview library with `go get`:

```
$ go get github.com/zserge/webview
```

Import the package and start using it:

```go
package main

import "github.com/zserge/webview"

func main() {
	// Open wikipedia in a 800x600 resizable window
	webview.Open("Minimal webview example",
		"https://en.m.wikipedia.org/wiki/Main_Page", 800, 600, true)
}
```

It is not recommended to use `go run` (although it works perfectly fine on Linux). Use `go build` instead:

```bash
# Linux
$ go build -o webview-example && ./webview-example

# MacOS uses app bundles for GUI apps
$ mkdir -p example.app/Contents/MacOS
$ go build -o example.app/Contents/MacOS/example
$ open example.app # Or click on the app in Finder

# Windows requires special linker flags for GUI apps.
# It's also recommended to use TDM-GCC-64 compiler for CGo.
# http://tdm-gcc.tdragon.net/download
$ go build -ldflags="-H windowsgui" -o webview-example.exe
```

### API

See [godoc](https://godoc.org/github.com/zserge/webview).

### How to serve or inject the initial HTML/CSS/JavaScript into the webview?

First of all, you probably want to embed your assets (HTML/CSS/JavaScript) into the binary to have a standalone executable. Consider using [go-bindata](https://github.com/jteeuwen/go-bindata) or any other similar tools.

Now there are two major approaches to deploy the content:

* Serve HTML/CSS/JS with an embedded HTTP server
* Injecting HTML/CSS/JS via JavaScript binding API

To serve the content it is recommended to use ephemeral ports:

```go
ln, err := net.Listen("tcp", "127.0.0.1:0")
if err != nil {
	log.Fatal(err)
}
defer ln.Close()
go func() {
 	// Set up your http server here
	log.Fatal(http.Serve(ln, nil))
}()
webview.Open("Hello", "http://"+ln.Addr().String(), 400, 300, false)
```

Injecting the content via JS bindings is a bit more complicated, but feels more solid and does not expose any additional open TCP ports.

Leave `webview.Settings.URL` empty to start with a bare minimal HTML5. It will open a webview with `<div id="app"></div>` in it. Alternatively, use data URI to inject custom HTML code (don't forget to URL-encode it):

```go
const myHTML = `<!doctype html><html>....</html>`
w := webview.New(webview.Settings{
	URL: `data:text/html,` + url.PathEscape(myHTML),
})
```

Keep your initial HTML short (a few kilobytes maximum).

Now you can inject more JavaScrtipt once the webview becomes ready using `webview.Eval()`. You can also inject CSS styles using JavaScript:

```go
w.Dispatch(func() {
	// Inject CSS
	w.Eval(fmt.Sprintf(`(function(css){
		var style = document.createElement('style');
		var head = document.head || document.getElementsByTagName('head')[0];
		style.setAttribute('type', 'text/css');
		if (style.styleSheet) {
			style.styleSheet.cssText = css;
		} else {
			style.appendChild(document.createTextNode(css));
		}
		head.appendChild(style);
	})("%s")`, template.JSEscapeString(myStylesCSS)))
	// Inject JS
	w.Eval(myJSFramework)
	w.Eval(myAppJS)
})
```

This works fairly well across the platforms, see `counter-go` example for more details about how make a webview app with no web server. It also demonstrates how to use ReactJS, VueJS or Picodom with webview.

### How to communicate between native Go and web UI?

You already have seen how to use `w.Eval()` to run Javascript inside the webview. There is also a way to call Go code from JavaScript.

On the low level there is a special callback, `webview.Settings.ExternalInvokeCallback` that receives a string argument. This string can be passed from JavaScript using `window.external.invoke_(someString)`.

This might seem very inconvenient, and that is why there is a dedicated `webview.Bind()` API call. It binds an existing Go object (struct or struct pointer) and creates/injects JS API for it. Now you can call JS methods and they will result in calling native Go methods. Even more, if you modify the Go object - it can be automatically serialized to JSON and passed to the web UI to keep things in sync.

Please, see `counter-go` example for more details about how to bind Go controllers to the web UI.

## Webview for C/C++ developers

### Getting started

Download [webview.h](https://raw.githubusercontent.com/zserge/webview/master/webview.h) and include it in your C/C++ code:

```c
// main.c
#include "webview.h"

#ifdef WIN32
int WINAPI WinMain(HINSTANCE hInt, HINSTANCE hPrevInst, LPSTR lpCmdLine,
                   int nCmdShow) {
#else
int main() {
#endif
  /* Open wikipedia in a 800x600 resizable window */
  webview("Minimal webview example",
	  "https://en.m.wikipedia.org/wiki/Main_Page", 800, 600, 1);
  return 0;
}
```

Build it:

```bash
# Linux
$ cc main.c -DWEBVIEW_GTK=1 $(shell pkg-config --cflags --libs gtk+-3.0 webkitgtk-3.0) -o webview-example
# MacOS
$ cc main.c -DWEBVIEW_COCOA=1 -x objective-c -framework Cocoa -framework WebKit -o webview-example
# Windows (mingw)
$ cc main.c -DWEBVIEW_WINAPI=1 -lole32 -lcomctl32 -loleaut32 -luuid -mwindows -o webview-example.exe
```

### API

For the most simple use cases there is only one function:

```c
int webview(const char *title, const char *url,	int width, int height, int resizable);
```

The following URL schemes are supported:

* `http://` and `https://`, no surprises here.
* `file:///` can be useful if you want to unpack HTML/CSS assets to some
  temporary directory and point a webview to open index.html from there.
* `data:text/html,<html>...</html>` allows to pass short HTML data inline
  without using a web server or pulluting the file system. Furhter
  modifications of the webview contents can be done via JavaScript bindings.

If have choosen a regular http URL scheme, you can use Mongoose or any other web server/framework you like.

If you want to have more control over the app lifecycle you can use the following functions:

```c
  struct webview webview = {
      .title = title,
      .url = url,
      .width = w,
      .height = h,
      .resizable = resizable,
  };
  /* Create webview window using the provided options */
  webview_init(&webview);
  /* Main app loop, can be either blocking or non-blocking */
  while (webview_loop(&webview, blocking) == 0);
  /* Destroy webview window, often exits the app */
  webview_exit(&webview);

  /* To change window title later: */
  webview_set_title(&webview, "New title");

  /* To terminate the webview main loop: */
  webview_terminate(&webview);
```

To evaluate arbitrary javascript code use the following C function:

```c
webview_eval(&webview, "alert('hello, world');");
```

There is also a special callback (`webview.external_invoke_cb`) that can be invoked from javascript:

```javascript
// C
void my_cb(struct webview *w, const char *arg) {
	...
}

// JS (note the trailing underscore)
window.external.invoke_('some arg');
// Exactly one string argument must be provided, to pass more complex objects
// serialize them to JSON and parse it in C. To pass binary data consider using
// base64.
window.external.invoke_(JSON.stringify({fn: 'sum', x: 5, y: 3}));
```

Webview library is meant to be used from a single UI thread only. So if you
want to call `webview_eval` or `webview_terminate` from some background thread
- you have to use `webview_dispatch` to post some arbitrary function with some
context to be executed inside the main UI thread:

```c
// This function will be executed on the UI thread
void render(struct webview *w, void *arg) {
  webview_eval(w, ......);
}

// Dispatch render() function from another thread:
webview_dispatch(w, render, some_arg);
```

## Notes

Execution on OpenBSD requires `wxallowed` [mount(8)](https://man.openbsd.org/mount.8) option.

## License

Code is distributed under MIT license, feel free to use it in your proprietary
projects as well.
