# Using custom builds of Godot

As you probably know, godot-rust interacts with Godot via the GDNative interface. This interface is formally specified in a file called `api.json`, which lists all the classes, functions, constants and other symbols. In its build step, godot-rust reads this file and generates Rust code reflecting the GDNative interface.

By default, godot-rust ships an `api.json` compatible with the latest Godot 3.x release. This makes it easy to use the latest version. But there are cases where you might want to use an older Godot version, or one that you built yourself (with custom compiler flags or modules). In the past, this needed quite a few manual steps; in the meantime, this process has been simplified.

For using custom Godot builds, the first thing you need to do is to add the feature flag `custom-godot` when adding godot-rust as a dependency.
For example, if you depend on the latest GitHub version of godot-rust, Cargo.toml would look like this:
```toml
gdnative = { git = "https://github.com/godot-rust/godot-rust.git", features = ["custom-godot"] }
```

Next, godot-rust must be able to locate the Godot engine executable on your system.  
There are two options:

1. Your executable is called `godot` and available in the system PATH.  
   On Windows systems, this would also find a `godot.bat`, for example.
2. You define an environment variable `GODOT_BIN` with the absolute path to your executable.  
   It is important that you include the filename -- this is not a directory path. 

That's it. During build, godot-rust will invoke Godot to generate a matching `api.json` -- you might see a short Godot window popping up.

Keep in mind that we only support Godot versions >= 3.2 and < 4.0 for now. Also, the GDNative API varies between versions, so you may need to adjust your client code. 


## Previous approach

> _**Note:** this guide is now obsolete._  
> _You can still use it when working with godot-rust 0.9 or `master` versions before December 2021._

Sometimes, users might need to use a different version of the engine that is different from the default one, or is a custom build. In order to use `godot-rust` with them, one would need to create a custom version of the `gdnative-bindings` crate, generated from an `api.json` from the custom build. This guide walks through the necessary steps to do so.

First, obtain the source code for `gdnative-bindings` from crates.io. For this guide, we'll use [`cargo-download`](https://github.com/Xion/cargo-download/) to accomplish this:

```
# Install the `cargo-download` sub-command if it isn't installed yet
cargo install cargo-download

# Download and unpack the crate
cargo download gdnative-bindings==0.9.0 >gdnative-bindings.tar.gz
tar -xf gdnative-bindings.tar.gz
```

You should be able to find the source code for the crate in a `gdnative-bindings-{version}` directory. Rename it to a distinctive name like `gdnative-bindings-custom` and update the `Cargo.toml`:

```toml
[package]
name = "gdnative-bindings-custom"
```

When downloading the crate, please specify the exact version of the crate that is specified in the `Cargo.toml` of `gdnative`. This is necessary because the generated bindings depend on internal interfaces that may change between non-breaking versions of `gdnative`.

After source is obtained, replace the API description file with one generated by your specific Godot build:

```
cd /path/to/gdnative-bindings-custom
/path/to/godot --gdnative-generate-json-api api.json

# Try to build and see if it works
cargo build
```

If everything goes well, you can now update the dependencies of your GDNative library to use this custom bindings crate:

```toml
[dependencies]

# Use the exact version corresponding to `gdnative-bindings`
# and disable the default re-export.
gdnative = { version = "=0.9.0", default-features = false, features = [] }

# Use your custom bindings crate as a path dependency
gdnative-bindings-custom = { path = "/path/to/gdnative-bindings-custom" }
```

Here, `gdnative` is specified using an exact version because the bindings generator is an internal dependency. When using custom binding crates, care must be taken to ensure that the version of the bindings crate used as the base matches the one specified in the `Cargo.toml` of the `gdnative` crate exactly, even for updates that are considered non-breaking in the `gdnative` crate. Using an exact version bound here helps prevent unintentional updates that may break the build.

Finally, replace references to `gdnative::api` with `gdnative-bindings-custom`. You should now be able to use the APIs in your custom build in Rust!


## Generating documentation

However, if you try to generate documentation with rustdoc at this point, you might notice that documentation might be missing or wrong for some of the types or methods. This is due to documentation being stored separately from the API description itself, and can be easily fixed if you have access to the source code from which your custom Godot binary is built.

To get updated documentation, you only need to copy all the documentation XMLs from `doc/classes` in the Godot source tree, to the `docs` directory in the `gdnative-bindings` source. After the files are copied, you should be able to get correct documentation for the API.