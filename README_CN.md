DOSBox ported to Emscripten
===========================

About
-----

[DOSBox](http://www.dosbox.com/) 是一个用来玩老游戏的开源dos模拟器。 [Emscripten](https://github.com/kripken/emscripten) 把C/C++代码转换为JavaScript代码。 这是一个可以用Emscripten编译的DOSBox版本，可以在Web浏览器中运行。它允许在Web浏览器中运行旧的DOS游戏和其他DOS程序。

DOSBOX是在GNU通用公共许可证下发布的。更多信息请看这里 [COPYING file](https://github.com/dreamlayers/em-dosbox/blob/em-dosbox-0.74/COPYING)

Status
------

EM-DOSBox可以在Web浏览器中成功运行大多数游戏。虽然DOSBox还没有完全重新构造为作为Emscripten主循环运行，但多亏了emterpreter sync，大多数功能都是可用的。由于分页异常，少数程序仍然会遇到问题。

Other issues
------------

* 游戏保存文件写入Emscripten文件系统，默认情况下，Emscripten文件系统是内存中的文件系统。关闭网页时，保存的游戏将丢失。
* 不支持在Windows中编译。由于使用GNU Autotools，构建过程需要类Unix环境。 See Emscripten
  [issue 2208](https://github.com/kripken/emscripten/issues/2208).
* Emscripten [issue 1909](https://github.com/kripken/emscripten/issues/1909)
曾经在多条件分之的情况下很低效。 现在看起来是修复了，但是Javascript V8引擎阻碍了多条件语句的优化。[issue 2275](http://code.google.com/p/v8/issues/detail?id=2275) 因此，原始的简单的内核被转换。x86指令的case语句变成函数，并使用函数指针数组代替switch语句。 使用`--enable-funarray`选项控制这个特性，默认是打开的。
* 浏览器的同源策略使得你没发使用 file:// 这样的URL。使用 `python -m SimpleHTTPServer` 这样的Web服务器来替代。
* In Firefox, ensure that
[dom.max\_script\_run\_time](http://kb.mozillazine.org/Dom.max_script_run_time)
 is set to a reasonable value that will allow you to regain control in case of
a hang.
* Firefox may use huge amounts of memory when starting asm.js builds which have
not been minified.
* The FPU code uses doubles and does not provide full 80 bit precision.
DOSBox can only give full precision when running on an x86 CPU.

Compiling
---------

Use the latest stable version of Emscripten (from the master branch). For
more information see the the
[Emscripten installation instructions](https://kripken.github.io/emscripten-site/docs/getting_started/downloads.html).
Em-DOSBox depends on bug fixes and new features found in recent versions of
Emscripten. Some Linux distributions have packages with old versions, which
should not be used.

First, create `./configure` by running `./autogen.sh`. Then
configure with `emconfigure ./configure` and build with `make`.
This will create `src/dosbox.js` which contains DOSBox and `src/dosbox.html`,
a web page for use as a template by the packager. These cannot be used as-is.
You need to provide DOSBox with files to run and command line arguments for
running them.

This branch supports SDL 2 and uses it by default. Emscripten will
automatically fetch SDL 2 from Emscripten Ports and build it. Use of `make -j`
to speed up compilation by running multiple Emscripten processes in parallel
[may break this](https://github.com/kripken/emscripten/issues/3033).
Once SDL 2 has been built by Emscripten, you can use `make -j`.
To use a different pre-built copy of Emscripten SDL 2, specify a path as in
`emconfigure ./configure --with-sdl2=/path/to/SDL-emscripten`. To use SDL 1,
give a `--with-sdl2=no` or `--without-sdl2` argument to `./configure`.

Emscripten emterpreter sync is used by default. This enables more DOSBox
features to work, but requires an Emscripten version after 1.29.4 and may
cause a small performance penalty. To disable use of emterpreter sync,
add the `--disable-sync` argument to `./configure`. When sync is used,
the Emscripten
[memory initialization file](https://kripken.github.io/emscripten-site/docs/optimizing/Optimizing-Code.html#memory-initialization)
is enabled, which means `dosbox.html.mem` needs to be in the same folder as
`dosbox.js`. The memory initialization file is large. Serve it in compressed
format to save bandwidth.

WebAssembly
-----------

[WebAssembly](https://github.com/kripken/emscripten/wiki/WebAssembly) is a
binary format which is meant to be a more efficient replacement for asm.js.
To build to WebAssembly instead of asm.js, the `-s WASM=1` option needs to
be added to the final `emcc` link command. There is no need to rebuild the
object files. You can do this using the `--enable-wasm` option when running
`./configure`. Since WebAssembly is still under very active development, use
the latest incoming Emscripten and browser development builds like
[Firefox Nightly](https://nightly.mozilla.org/) and
[Chrome Canary](https://www.google.com/chrome/browser/canary.html).

Building with WebAssembly changes the dosbox.html file. Files packaged
using a dosbox.html without WebAssembly will not work with a WebAssembly
build.

Running DOS Programs
--------------------

To run DOS programs, you need to provide a suitable web page, load files into
the Emscripten file system, and give command line arguments to DOSBox. The
simplest method is by using the included packager tools.

The normal packager tool is `src/packager.py`, which runs the Emscripten
packager. It requires `dosbox.html`, which is created when building Em-DOSBox.
If you do not have Emscripten installed, you need to use `src/repackager.py`
instead. Any packager or repackager HTML output file can be used as a template
for the repackager. Name it `template.html` and put it in the same directory
as `repackager.py`.

The following instructions assume use of the normal packager. If using
repackager, replace `packager.py` with `repackager.py`. You need
[Python 2](https://www.python.org/downloads/) to run either packager.

If you have a single DOS executable such as `Gwbasic.exe`, place
it in the same `src` directory as `packager.py` and package it using:

```./packager.py gwbasic Gwbasic.exe```

This creates `gwbasic.html` and `gwbasic.data`. Placing those in the same
directory as `dosbox.js` and viewing `gwbasic.html` will run the program in a
web browser:

Some browsers have a same origin policy that prevents access to the required
data files while using `file://` URLs. To get around this you can use Python's
built in Really Simple HTTP Server and point the browser to
[http://localhost:8000](http://localhost:8000).

```python -m SimpleHTTPServer 8000```

If you need to package a collection of DOS files. Place all the files in a
single directory and package that directory with the executable specified. For
example, if Major Stryker's files are in the subdirectory `src/major_stryker`
and it's launched using `STRYKER.EXE` you would package it using:

```./packager.py stryker major_stryker STRYKER.EXE```

Again, place the created `stryker.html` and `stryker.data` files in the same
directory as `dosbox.js` and view `stryker.html` to run the game in browser.

You can also include a [DOSBox
configuration](http://www.dosbox.com/wiki/Dosbox.conf) file that will be
acknowledged by the emulator to modify any speed, audio or graphic settings.
Simply include a `dosbox.conf` text file in the package directory before you
run `./packager.py`.

To attempt to run Major Stryker in CGA graphics mode, you would create the
configuration file `/src/major_stryker/dosbox.conf` and include this body of
text:

```
[dosbox]
machine=cga
```

Then package it using:

```./packager.py stryker-cga major_stryker STRYKER.EXE```

Credits
-------

Most of the credit belongs to the
[DOSBox crew](http://www.dosbox.com/crew.php).
They created DOSBox and made it compatible with a wide variety of DOS games.
[Ismail Khatib](https://github.com/CeRiAl) got DOSBox
to compile with Emscripten, but didn't get it to work.
[Boris Gjenero](https://github.com/dreamlayers)
started with that and got it to work. Then, Boris re-implemented
Ismail's changes a cleaner way, fixed issues and improved performance to make
many games usable in web browsers. Meanwhile,
[Alon Zakai](https://github.com/kripken/) quickly fixed Emscripten bugs which
were encountered and added helpful Emscripten features.
