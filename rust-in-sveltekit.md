# How to do WebAssembly with Rust in a SvelteKit project

In this blog post I'm going to take you on a journey where I tried to connect WebAssembly with Rust to my Sveltekit project. I made this journey in December of last year (as of this writing) when I was doing some code challenges with [Advent of Code](https://adventofcode.com/) and decided to make a [Sveltekit website](https://adventofcode2022-tice-sveltekit.netlify.app/) and make them in there. Initially I decided for Javascript (Typescript), a language I'm very familiar with as I've been developing with it for 4 years professionally.

But after four or five days I was in for a new challenge. And [WebAssembly (Wasm)](https://webassembly.org/) has always been on my radar. I was quite amazed at how it worked and how it could bring high performant code with a low-level implementation to the web browser.

So I started digging in. I mean, other than what I heard from it and what it tries to solve, I had no idea how it worked and how it would actually be implemented in the browser (I even thought it was just a "portal" to running something like Rust right in your browser, but it's not really like that).

## Wasm-pack

So I started my journey with good-old Youtube tutorials. And they all kind of seemed to be doing the same thing: compiling Rust code to WebAssembly with the help of [wasm-pack](https://rustwasm.github.io/wasm-pack/). Wasm-pack is a tool that helps with compiling your Rust code to a WebAssembly package that you can then register with something like NPM, after which it is just seen as a package you can import its functions from. 

So I started out creating a new Rust library. The way this was done was with the help of [`cargo`](https://crates.io/) which is the package manager for Rust (as Rust calls its packages crates, crates are cargo, there you go). By running `cargo new --lib <name>` it would create a new folder with the name you give it, with in there a couple files.

First of all there is the `Cargo.toml` file. This is basically like the `package.json` for Rust. In order to work with WebAssembly, we need to add a package here called `wasm-bindgen`. My `Cargo.toml` looked like this:

```toml
[package]
name = "myfirstwasm"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

What I found interesting is that you had to specify the version number in the `toml` file (literally write it out) and then run `cargo install` which would globally install the dependency. You can always find the latest version at [crates.io](https://crates.io/) but I still found that a little odd. Makes me wonder how easy it is to upgrade packages... ah well that's for later concern.

Then in the `src` folder there is a `lib.rs` file, and this is where I wrote my first Rust function, which was a simple `add` function:

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

To make it so that wasm-pack can "read" the function and expose it with the same name as you give it here in your NPM package, you have to add a little `#[wasm_bindgen]` decorator above the function.

After this it's just a matter of running `wasm-pack build --target web` in the terminal and it would create a `pkg` folder right next to the `src` folder.

In here a couple of files are living. I briefly took the time to take a look at them. There is a `js` file with the name of your project, which is the entry way for the package. Here (I think) it is making the connection between it and the `<name>_bg.wasm` file, which contains the Wasm binary. It's quite a large file! Also I was glad to see Typescript declaration files. Rust is a typed language (I'd even call it Typescript on steriods in terms of strictness) so these types are translated to Typescript types as well. For example for my `add` function, it expects 2 arguments which are both of type `i32` (32-bit integer) in Rust, which just translates to `number` type in Typescript. Good stuff! Lastly there was a `.gitignore` in there as well, handy dandy as you don't want this code to pollute your git repo (although I'll get back to that at the deploy step 😉).

The wasm tutorial ended with making an `index.html` file, and importing the script directly with the help of browsers' hot and happening module support:

```html
<script type="module">
  import init, { add } from './pkg/myfirstwasm.js'

  await init()

  console.log(add(1, 2))
</script>
```

Fair enough, it worked great like this! (Don't forget to await `init()` function which is default export from the package, which loads the wasm module) But this is not how I'm used to handle dependencies, of course we are using NPM. Plus I wanted to make this work in my Sveltekit website which uses NPM as well of course. So let's get going!

## Registering the Wasm NPM package

So after this, I moved over the code to my Sveltekit project. I could've put it in the `src/lib` folder in Sveltekit, but I decided to create a folder next to it called `src/rust`. I later [found out](https://www.reddit.com/r/sveltejs/comments/vzf86d/comment/ikaza65/?utm_source=share&utm_medium=web2x&context=3) it was important this code at least lived in the `src` folder, otherwise it would've posed problems with `vite`.

To "register" it as an NPM package, simply add it to your `dependencies` list in package.json like this:

```json
"<packageName>": "./path/to/the/rust/pkg"
```

After hitting `npm i` your package is ready for use in your project!

## Pitfalls with Sveltekit

So yeah I was very glad I found [this Reddit comment](https://www.reddit.com/r/sveltejs/comments/vzf86d/comment/ikaza65/?utm_source=share&utm_medium=web2x&context=3) as after following its advice, my WebAssembly package was all working like a charm in my SvelteKit project.

Basically, what I had to do was install 2 extra packages: `vite-plugin-wasm` and `vite-plugin-top-level-await`. And of course make sure that my Rust crate (i.e. folder with `Cargo.toml` and Rust code) was living inside of the `src` folder. Then importing from the package worked like a charm! See my SvelteKit page for an example with loaded in Wasm:

```html
<script lang="ts">
  import init, { add } from 'rust-package'

  let result: number

  onMount(async () => {
    await init()

    result = add(1, 2)
  })
</script>

<p>{result}</p>
```

## Getting it into production

Now I think the hardest part started. Because how would I integrate this into my build project hosted on Netlify? Now technically with all the output created, it could easily live on Netlify. As it'd just be another installed NPM package with compiled Wasm code, that could be loaded in the browser. The main problem was getting that code to compile in Netlify.

Now surprisingly, I couldn't find any information on compiling Rust code as part of the build process on Netlify, zero. I had to do things myself. I did read here Netlify had Rust installed by default, but adding something like `wasm-build --target web` on like the `postinstall` hook in `package.json` wouldn't work at all. It just wouldn't find `wasm-build` and it wouldn't be able to install.

Now had I been able to deploy with for example a `Dockerfile` things would've been easier. Because then I could choose all the dependencies I wanted and could really start with a clean slate. I could install Rust and Node seperately and have all the things I needed. For example I could do something like this with [Railway](https://railway.app/) which is kind of a successor to Heroku since they got rid of their free tier. Stay tuned for a blog post where I go through migrating my Heroku projects to Railway!

Anyways, as I didn't want to host my website anywhere else for the time being, I decided on doing something else. I decided to make a pre-commit hook, which would compile my Rust code on commit, and would handily remove the auto-generated `.gitignore` file and add the changes to git. A bit of a nasty way of doing it 😅 and it would make git diffs a bit dirtier as it would contain build output code... but ah well I'm the only one working on this project anyways.

As I'd never done pre-commit hooks before, I found the package [pre-commit](https://www.npmjs.com/package/pre-commit) on NPM, assuming the name would say what it did I installed it in a heartbeat and followed the readme. But for the life of me I couldn't get this thing to work!! It was a pain. Because there was no feedback, it just didn't work. I committed, and nothing happened.

I don't know exactly how I got to this package, but I stumbled upon [husky](https://www.npmjs.com/package/husky) which would make use of the native git hooks rather than overwrite them (look at the enormous difference in weekly downloads). Followed the readme install guide again and it worked like a charm. Too bad for the not-so-clear naming, but for the rest all the praise goes to Husky.

My pre-commit hook code (in `.husky/pre-commit`) ended up looking like this:

```bash
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm rust:build
pnpm remove-rust-gitignore
```

_Note_: I was using `pnpm` for this project, but if you don't fancy the new awesome package manager tool, then just replace it with `npm` or `yarn` 😉

Of course these 2 commands were pointing to 2 scripts in `package.json` which were the following:

```json
"scripts": {
  ...
  "rust:build": "wasm-pack build src/rust --target web",
  "remove-rust-gitignore": "rm src/rust/pkg/.gitignore && git add src/rust/pkg",
  ...
}
```

This ended up working really well. It's only a bit annoying when you're not making any changes in Rust code, you will still recompile the Rust code on every commit, and also it assumes you have Rust + `wasm-pack` installed on your pc (but as it's my personal project I wasn't too worried about that).

## Optimizing the development workflow

So it's almost almost perfect. We got a nice build flow, and as we're using SvelteKit, we're pretty happy by default. There was only one thing: I'd have to recompile the Rust code with the command if I wanted to see my changes on the page (I didn't have to refresh the page, as hot-reloading picks up the package changes in SvelteKit). So I went on a search for a Rust watcher that would compile my code every time I made a change to the file. And strangely enough `wasm-pack` doesn't provide a watch command by itself, but it is possible with `cargo watch` command. I ended up with a script I could call with `pnpm rust:dev` which looked something like this:

```json
"scripts": {
  ...
  "rust:dev": "cd src/rust && cargo watch -i .gitignore -i \"pkg/*\" -s \"wasm-pack build --target web\"",
  ...
}
```

So it'd `cd` into the rust crate folder, and watch the files, ignoring `.gitignore` and the build output folder `pkg`. Great stuff!

And of course, as developers we are lazy, I made a `dev:all` command, which would start an [overmind](https://github.com/DarthSim/overmind) process with the SvelteKit dev server and this watcher, the `Procfile` ended up like this:

```
web: pnpm dev
rust: pnpm rust:dev
```

But yeah this is of course open to be implemented however you like.

## Conclusion

It took me pretty much a full day to get all of this to work. It was a lot of fun though, and finally seeing the WebAssembly code executing blazingly fast in my production build was definitely very rewarding to see. I learnt a lot about Wasm and how it actually worked. Unfortunately I haven't found an actual use case where I could really start making a Wasm project or function for that would otherwise be too cpu-intensive in order to work. But as I know and heard from my colleagues it is used for things like virtual backgrounds for webcams and stuff like that, so who knows I might be building something like that some day.

For now I'll try to do my Advent of Code challenges (starting day 6) in Rust! It's not gonna be easy... I already got a taste of it with day 6... yeah it's pretty hard haha. Anyways do expect another blog post about that when I learn a bit more about the ins and outs of Rust 😄.

## Resources

SvelteKit Advent of Code project with Wasm implementation [website](https://adventofcode2022-tice-sveltekit.netlify.app/) and [repository](https://github.com/TravelingTice/advent_of_code_2022_sveltekit).