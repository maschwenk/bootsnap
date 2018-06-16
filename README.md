# Bootsnap [![Build Status](https://travis-ci.org/Shopify/bootsnap.svg?branch=master)](https://travis-ci.org/Shopify/bootsnap)

Bootsnap is a library that plugs into Ruby, with optional support for `ActiveSupport` and `YAML`,
to optimize and cache expensive computations. See [How Does This Work](#how-does-this-work).

#### Performance
- [Discourse](https://github.com/discourse/discourse) reports a boot time reduction of approximately 50%, from roughly 6 to 3 seconds on one machine;
- One of our smaller internal apps also sees a reduction of 50%, from 3.6 to 1.8 seconds;
- The core Shopify platform -- a rather large monolithic application -- boots about 75% faster, dropping from around 25s to 6.5s.

## Usage

This gem works on MacOS and Linux.

Add `bootsnap` to your `Gemfile`:

```ruby
gem 'bootsnap', require: false
```

If you are using Rails, add this to `config/boot.rb` immediately after require `bundler/setup` or a `Bundler.setup` call:
```ruby
 require 'bundler/setup'
 require 'bootsnap/setup'
```
Protip: Using Bundler.setup like in the example below can _further_ improve boot time by reducing the `$LOAD_PATH` to only include gems for the given environment
```ruby
require 'bundler'
Bundler.setup(:default, ENV.fetch('RAILS_ENV', :development))
require 'bootsnap/setup'
```

You can see how this require works [here](https://github.com/Shopify/bootsnap/blob/master/lib/bootsnap/setup.rb).

If you are not using Rails, or if you are but want more control over things, add this to your
application setup immediately after `require 'bundler/setup'` (i.e. as early as possible: the sooner
this is loaded, the sooner it can start optimizing things)

```ruby
require 'bootsnap'
env = ENV['RAILS_ENV'] || "development"
Bootsnap.setup(
  cache_dir:            'tmp/cache',          # Path to your cache
  development_mode:     env == 'development', # Current working environment, e.g. RACK_ENV, RAILS_ENV, etc
  load_path_cache:      true,                 # Optimize the LOAD_PATH with a cache
  autoload_paths_cache: true,                 # Optimize ActiveSupport autoloads with cache
  disable_trace:        true,                 # (Alpha) Set `RubyVM::InstructionSequence.compile_option = { trace_instruction: false }`
  compile_cache_iseq:   true,                 # Compile Ruby code into ISeq cache, breaks coverage reporting.
  compile_cache_yaml:   true                  # Compile YAML into a cache
)
```

**Protip:** You can replace `require 'bootsnap'` with `BootLib::Require.from_gem('bootsnap',
'bootsnap')` using [this trick](https://github.com/Shopify/bootsnap/wiki/Bootlib::Require). This
will help optimize boot time further if you have an extremely large `$LOAD_PATH`.

Note: Bootsnap and [Spring](https://github.com/rails/spring) are orthogonal tools. While Bootsnap
speeds up the loading of individual source files, Spring keeps a copy of a pre-booted Rails process
on hand to completely skip parts of the boot process the next time it's needed. The two tools work
well together, and are both included in a newly-generated Rails applications by default.

### Environments

All Bootsnap features are enabled in development, test, production, and all other environments according to the configuration in the setup. At Shopify, we use this gem safely in all environments without issue.

If you would like to disable any feature for a certain environment, we suggest changing the configuration to take into account the appropriate ENV var or configuration according to your needs.

## How does this work?

Bootsnap optimizes methods to cache results of expensive computations, and can be grouped
into two broad categories:

* [Path Pre-Scanning](#path-pre-scanning)
    * `Kernel#require` and `Kernel#load` are modified to eliminate `$LOAD_PATH` scans.
    * `ActiveSupport::Dependencies.{autoloadable_module?,load_missing_constant,depend_on}` are
      overridden to eliminate scans of `ActiveSupport::Dependencies.autoload_paths`.
* [Compilation caching](#compilation-caching)
    * `RubyVM::InstructionSequence.load_iseq` is implemented to cache the result of ruby bytecode
      compilation.
    * `YAML.load_file` is modified to cache the result of loading a YAML object in MessagePack format
      (or Marshal, if the message uses types unsupported by MessagePack).

### Path Pre-Scanning

*(This work is a minor evolution of [bootscale](https://github.com/byroot/bootscale)).*

Upon initialization of bootsnap or modification of the path (e.g. `$LOAD_PATH`),
`Bootsnap::LoadPathCache` will fetch a list of requirable entries from a cache, or, if necessary,
perform a full scan and cache the result.

Later, when we run (e.g.) `require 'foo'`, ruby *would* iterate through every item on our
`$LOAD_PATH` `['x', 'y', ...]`,  looking for `x/foo.rb`, `y/foo.rb`, and so on. Bootsnap instead
looks at all the cached requirables for each `$LOAD_PATH` entry and substitutes the full expanded
path of the match ruby would have eventually chosen.

If you look at the syscalls generated by this behaviour, the net effect is that what would
previously look like this:

```
open  x/foo.rb # (fail)
# (imagine this with 500 $LOAD_PATH entries instead of two)
open  y/foo.rb # (success)
close y/foo.rb
open  y/foo.rb
...
```

becomes this:

```
open y/foo.rb
...
```

Exactly the same strategy is employed for methods that traverse
`ActiveSupport::Dependencies.autoload_paths` if the `autoload_paths_cache` option is given to
`Bootsnap.setup`.

The following diagram flowcharts the overrides that make the `*_path_cache` features work.

![Flowchart explaining
Bootsnap](https://cloud.githubusercontent.com/assets/3074765/24532120/eed94e64-158b-11e7-9137-438d759b2ac8.png)

Bootsnap classifies path entries into two categories: stable and volatile. Volatile entries are
scanned each time the application boots, and their caches are only valid for 30 seconds. Stable
entries do not expire -- once their contents has been scanned, it is assumed to never change.

The only directories considered "stable" are things under the Ruby install prefix
(`RbConfig::CONFIG['prefix']`, e.g. `/usr/local/ruby` or `~/.rubies/x.y.z`), and things under the
`Gem.path` (e.g. `~/.gem/ruby/x.y.z`) or `Bundler.bundle_path`. Everything else is considered
"volatile".

In addition to the [`Bootsnap::LoadPathCache::Cache`
source](https://github.com/Shopify/bootsnap/blob/master/lib/bootsnap/load_path_cache/cache.rb),
this diagram may help clarify how entry resolution works:

![How path searching works](https://cloud.githubusercontent.com/assets/3074765/25388270/670b5652-299b-11e7-87fb-975647f68981.png)


It's also important to note how expensive `LoadError`s can be. If ruby invokes
`require 'something'`, but that file isn't on `$LOAD_PATH`, it takes `2 *
$LOAD_PATH.length` filesystem accesses to determine that. Bootsnap caches this
result too, raising a `LoadError` without touching the filesystem at all.

### Compilation Caching

*(A more readable implementation of this concept can be found in
[yomikomu](https://github.com/ko1/yomikomu)).*

Ruby has complex grammar and parsing it is not a particularly cheap operation. Since 1.9, Ruby has
translated ruby source to an internal bytecode format, which is then executed by the Ruby VM. Since
2.3.0, Ruby [exposes an API](https://ruby-doc.org/core-2.3.0/RubyVM/InstructionSequence.html) that
allows caching that bytecode. This allows us to bypass the relatively-expensive compilation step on
subsequent loads of the same file.

We also noticed that we spend a lot of time loading YAML documents during our application boot, and
that MessagePack and Marshal are *much* faster at deserialization than YAML, even with a fast
implementation. We use the same strategy of compilation caching for YAML documents, with the
equivalent of Ruby's "bytecode" format being a MessagePack document (or, in the case of YAML
documents with types unsupported by MessagePack, a Marshal stream).

These compilation results are stored in a cache directory, with filenames generated by taking a hash
of the full expanded path of the input file (FNV1a-64).

Whereas before, the sequence of syscalls generated to `require` a file would look like:

```
open    /c/foo.rb -> m
fstat64 m
close   m
open    /c/foo.rb -> o
fstat64 o
fstat64 o
read    o
read    o
...
close   o
```

With bootsnap, we get:

```
open      /c/foo.rb -> n
fstat64   n
close     n
open      /c/foo.rb -> n
fstat64   n
open      (cache) -> m
read      m
read      m
close     m
close     n
```

This may look worse at a glance, but underlies a large performance difference.

*(The first three syscalls in both listings -- `open`, `fstat64`, `close` -- are not inherently
useful. [This ruby patch](https://bugs.ruby-lang.org/issues/13378) optimizes them out when coupled
with bootsnap.)*

Bootsnap writes a cache file containing a 64 byte header followed by the cache contents. The header
is a cache key including several fields:

* `version`, hardcoded in bootsnap. Essentially a schema version;
* `os_version`, A hash of the current kernel version (on macOS, BSD) or glibc version (on Linux);
* `compile_option`, which changes with `RubyVM::InstructionSequence.compile_option` does;
* `ruby_revision`, the version of Ruby this was compiled with;
* `size`, the size of the source file;
* `mtime`, the last-modification timestamp of the source file when it was compiled; and
* `data_size`, the number of bytes following the header, which we need to read it into a buffer.

If the key is valid, the result is loaded from the value. Otherwise, it is regenerated and clobbers
the current cache.

### Putting it all together

Imagine we have this file structure:

```
/
├── a
├── b
└── c
    └── foo.rb
```

And this `$LOAD_PATH`:

```
["/a", "/b", "/c"]
```

When we call `require 'foo'` without bootsnap, Ruby would generate this sequence of syscalls:


```
open    /a/foo.rb -> -1
open    /b/foo.rb -> -1
open    /c/foo.rb -> n
close   n
open    /c/foo.rb -> m
fstat64 m
close   m
open    /c/foo.rb -> o
fstat64 o
fstat64 o
read    o
read    o
...
close   o
```

With bootsnap, we get:

```
open      /c/foo.rb -> n
fstat64   n
close     n
open      /c/foo.rb -> n
fstat64   n
open      (cache) -> m
read      m
read      m
close     m
close     n
```

If we call `require 'nope'` without bootsnap, we get:

```
open    /a/nope.rb -> -1
open    /b/nope.rb -> -1
open    /c/nope.rb -> -1
open    /a/nope.bundle -> -1
open    /b/nope.bundle -> -1
open    /c/nope.bundle -> -1
```

...and if we call `require 'nope'` *with* bootsnap, we get...

```
# (nothing!)
```
