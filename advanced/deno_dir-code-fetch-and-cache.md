---
description: >-
  In this section, we will discuss DENO_DIR, a folder that caches compiled or
  remote files, REPL history, etc. It is to some extent similar to GOPATH of Go.
  We will also check out code fetch logic.
---

# DENO\_DIR, Code Fetch and Cache

## DENO\_DIR Structure

`DENO_DIR` contains the following files and directories:

```text
# DIRECTORIES
gen/: Cache for files compiled to JavaScript
deps/: Cache for remote url imported files
  |__ http/: For http imports
  |__ https/: For https imports

# FILES
deno_history.txt: History of Deno REPL
```

By default, `DENO_DIR` is located in `$HOME/.deno`. However, the user could also change its location by modifying the `$DENO_DIR` environment variable. Explicitly setting `DENO_DIR` is recommended in production.

### gen/

`$DENO_DIR/gen/` is used to store JavaScript files that are compiled from source TypeScript files. Such compilation is necessary since V8 does not recognize TypeScript syntax beyond the JS subset.

Each JS file inside of the `gen/` folder has a filename that is the hash of its TypeScript source. It also comes with a source map ending with `.map`.

This cache exists to avoid repeated recompilation each run while the source file is not updated by the user. For example, suppose we have a file `hello-world.ts` which contains nothing but `console.log("Hello world")`. During the first run, we will see the message about compilation:

```text
$ deno hello-world.ts
Compiling /Users/kevinqian/my-folder/hello-world.ts
Hello world
```

However, without changing the file, when you rerun the code:

```text
$ deno hello-world.ts
Hello world
```

The compilation message is gone. This is because for this run, instead of compiling again, Deno is directly using the cached version in `gen/`.

The cache load and save code could be found in `DenoDir::load_cache` and `DenoDir::code_cache`, from `src/deno_dir.rs`.

To force Deno to recompile your TypeScript code instead of using the cached version, you could use the `--recompile` flag.

### deps/

`$DENO_DIR/deps` is used to store files fetched through remote url import. It contains subfolders based on url scheme \(currently only `http` and `https`\), and store files to locations based on the URL path. For example, for the following import \(notice that Deno requires the user to specify extensions explicitly\):

```typescript
import { serve } from "https://deno.land/x/std/net/http.ts";
```

The downloaded `http.ts` will be locally stored in

```text
$DENO_DIR/deps/https/deno.land/x/std/net/http.ts
```

Notice that unless the user run the code using `--reload` flag, `http.ts` in our case would NEVER be re-downloaded in future runs.

Currently \(**WARNING: MIGHT BE CHANGED IN THE FUTURE**\), Deno would keep an eye on the content MIME type of downloaded remote files. In the cases where a file is missing an extension, or having an extension that does not match the content type, Deno would create an extra file ended with `.mime` to store the MIME type as provided by HTTP response headers. Therefore, if we download a file named `a.ts` while having `Content-Type: text/javascript` in the response header, a file `a.ts.mime` would be created to its side, containing `text/javascript`. Due to the presence of this `.mime` file, `a.ts` would later be imported as if it is a JavaScript file.

## Code Fetch Logic

Let's now look into how Deno resolves the code for an import.

Go to `src/deno_dir.rs` and find the following function:

{% code-tabs %}
{% code-tabs-item title="src/deno\_dir.rs" %}
```rust
  fn get_source_code(
    self: &Self,
    module_name: &str,
    filename: &str,
  ) -> DenoResult<CodeFetchOutput> {
    let is_module_remote = is_remote(module_name);
    // We try fetch local. Two cases:
    // 1. This is a remote module, but no reload provided
    // 2. This is a local module
    if !is_module_remote || !self.reload {
      debug!(
        "fetch local or reload {} is_module_remote {}",
        module_name, is_module_remote
      );
      match self.fetch_local_source(&module_name, &filename)? {
        Some(output) => {
          debug!("found local source ");
          return Ok(output);
        }
        None => {
          debug!("fetch_local_source returned None");
        }
      }
    }

    // If not remote file, stop here!
    if !is_module_remote {
      debug!("not remote file stop here");
      return Err(DenoError::from(std::io::Error::new(
        std::io::ErrorKind::NotFound,
        format!("cannot find local file '{}'", filename),
      )));
    }

    debug!("is remote but didn't find module");

    // not cached/local, try remote
    let maybe_remote_source =
      self.fetch_remote_source(&module_name, &filename)?;
    if let Some(output) = maybe_remote_source {
      return Ok(output);
    }
    return Err(DenoError::from(std::io::Error::new(
      std::io::ErrorKind::NotFound,
      format!("cannot find remote file '{}'", filename),
    )));
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

 `module_name` is a string representing the module to be loaded, created based on the paths of the path of the file to be imported, and the file that contains the `import` statement. Therefore, `import "./b.ts"` inside of the file imported by `import "https://example.com/a.ts"` would be treated as if it is `import "https://example.com/b.ts"`.

`fetch_local_source` goes into the filesystem and try resolving the content based on the filename and its type:

{% code-tabs %}
{% code-tabs-item title="src/deno\_dir.rs" %}
```rust
fn fetch_local_source(
    self: &Self,
    module_name: &str,
    filename: &str,
  ) -> DenoResult<Option<CodeFetchOutput>> {
    let p = Path::new(&filename);
    let media_type_filename = [&filename, ".mime"].concat();
    let mt = Path::new(&media_type_filename);
    let source_code = match fs::read(p) {
      Err(e) => {
        if e.kind() == std::io::ErrorKind::NotFound {
          return Ok(None);
        } else {
          return Err(e.into());
        }
      }
      Ok(c) => c,
    };
    // .mime file might not exists
    let maybe_content_type_string = fs::read_to_string(&mt).ok();
    // Option<String> -> Option<&str>
    let maybe_content_type_str =
      maybe_content_type_string.as_ref().map(String::as_str);
    Ok(Some(CodeFetchOutput {
      module_name: module_name.to_string(),
      filename: filename.to_string(),
      media_type: map_content_type(&p, maybe_content_type_str),
      source_code,
      maybe_output_code: None,
      maybe_source_map: None,
    }))
  }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

while `fetch_remote_source` goes and downloads the remote file and caches it in `$DENO_DIR/deps`, possibly also creating the `.mime` files:

```rust
fn fetch_remote_source(
    self: &Self,
    module_name: &str,
    filename: &str,
  ) -> DenoResult<Option<CodeFetchOutput>> {
    let p = Path::new(&filename);
    // We write a special ".mime" file into the `.deno/deps` directory along side the
    // cached file, containing just the media type.
    let media_type_filename = [&filename, ".mime"].concat();
    let mt = Path::new(&media_type_filename);
    eprint!("Downloading {}...", &module_name); // no newline
    let maybe_source = http_util::fetch_sync_string(&module_name);
    if let Ok((source, content_type)) = maybe_source {
      eprintln!(""); // next line
      match p.parent() {
        Some(ref parent) => fs::create_dir_all(parent),
        None => Ok(()),
      }?;
      deno_fs::write_file(&p, &source, 0o666)?;
      // Remove possibly existing stale .mime file
      let _ = std::fs::remove_file(&media_type_filename);
      // Create .mime file only when content type different from extension
      let resolved_content_type = map_content_type(&p, Some(&content_type));
      let ext = p
        .extension()
        .map(|x| x.to_str().unwrap_or(""))
        .unwrap_or("");
      let media_type = extmap(&ext);
      if media_type == msg::MediaType::Unknown
        || media_type != resolved_content_type
      {
        deno_fs::write_file(&mt, content_type.as_bytes(), 0o666)?
      }
      return Ok(Some(CodeFetchOutput {
        module_name: module_name.to_string(),
        filename: filename.to_string(),
        media_type: map_content_type(&p, Some(&content_type)),
        source_code: source,
        maybe_output_code: None,
        maybe_source_map: None,
      }));
    } else {
      eprintln!(" NOT FOUND");
    }
    Ok(None)
  }
```

Thus, the resolution logic goes as follows:

* If `module_name` starts with a remote url scheme:
  * If `--reload` flag is present, force download the file and use it.
  * Otherwise
    * If the local cached file is present, use it.
    * Otherwise, download the file to `$DENO_DIR/deps` and use it.
* If `module_name` represents a local source, use the local file.





