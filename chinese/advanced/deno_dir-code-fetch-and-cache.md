---
描述: >-
  在这一部分，我们将会讨论DENO_DIR，这是一个缓存编译后的或者远程的文件，REPL历史记录等等的目录。他跟Go的GOPATH是一样的。我们也将会讨论代码获取逻辑。
---

# DENO\_DIR, Code Fetch 和 Cache

## DENO\_DIR 结构

`DENO_DIR`包含下面的文件和目录：

```text
# DIRECTORIES
gen/: 缓存编译为JavaScript的文件
deps/: 缓存导入的远程url的文件
  |__ http/: http方式导入的文件
  |__ https/: https方式导入的文件

# FILES
deno_history.txt: Deno REPL历史记录缓存
```

默认的，`DENO_DIR`位于`$HOME/.deno`。但是，用户可以通过修改`$DENO_DIR`环境变量来修改该位置。在生产环境建议显式的设置`DENO_DIR`。

### gen/

`$DENO_DIR/gen/`被用来存放JavaScript文件，这些文件是从TypeScript源码编译来的。这样的编译是必要的，因为V8不识别JS子集之外的TypeScript语法。

`gen/`目录下的每一个JS文件的文件名是他的TypeScript源码的hash值。同时JS文件也对应一个`.map`为后缀的source map文件。

缓存存在的原因是为了避免在用户没有修改代码的情况下，每次运行时不断的重新编译文件。比如我们有一个`hello-world.ts`文件，他只是包含了代码`console.log("Hello world")`。在第一次运行时，我们会看到编译信息：


```text
$ deno hello-world.ts
Compiling /Users/kevinqian/my-folder/hello-world.ts
Hello world
```

但是在没有修改文件内容的情况下，当你重新运行代码：

```text
$ deno hello-world.ts
Hello world
```

不会再有编译信息的提示。这是因为在这一次运行中，Deno直接使用了`gen/`中缓存版本，而不是再次编译。

缓存加载和保存的代码，可以从文件`src/deno_dir.rs`中的`DenoDir::load_cache`和`DenoDir::code_cache`中找到。

为了强制Deno重新编译你的代码而不是使用缓存的版本，你需要使用`--recompile`标志。 

### deps/

`$DENO_DIR/deps`被用来保存远端url import 获得的文件。根据url的模式，他包含了子目录\(现在只有`http`和`https`\)，并且保存文件的位置由URL path决定。比如，对于下面的的import\(请注意，Deno要求用户显式地指定扩展\)。

```typescript
import { serve } from "https://deno.land/x/std/net/http.ts";
```

下载的`http.ts`文件将会被存储在：

```text
$DENO_DIR/deps/https/deno.land/x/std/net/http.ts
```

需要注意，除非用户用`--reload`标志运行代码，否则我们的`http.ts`文件在接下来的运行中不会被重新下载。

当前\(**警告：将来可能改变**\)，Deno会关注从远端下载的文件的内容的MIME类型。在文件缺少扩展名或扩展名与内容类型不匹配的情况下，Deno将创建一个以`.mime`结尾的额外文件，来存储HTTP响应头提供的mime类型。如果我们下载的文件名是`a.ts`，然而响应头里面是`Content-Type: text/javascript`，一个包含`text/javascript`内容的`a.ts.mime`文件将会在他旁边被创建。由于`.mime`文件的存在，`a.ts`后面将会被当做一个JavaScript文件被import。

## 代码下载逻辑

现在让我们看看Deno如何解析import的代码。

进入`src/deno_dir.rs`文件，然后找到下面的函数：

{% code-tabs %}
{% code-tabs-item title="src/deno\_dir.rs" %}
```rust
  fn get_source_code(
    self: &Self,
    module_name: &str,
    filename: &str,
  ) -> DenoResult<CodeFetchOutput> {
    let is_module_remote = is_remote(module_name);
    // 我们尝试加载本地文件. 两个例子:
    // 1. 这是一个远程的模块, 但是没有提供reload标志
    // 2. 这是一个本地的模块
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

    // 如果不是远程文件，停止运行！
    if !is_module_remote {
      debug!("not remote file stop here");
      return Err(DenoError::from(std::io::Error::new(
        std::io::ErrorKind::NotFound,
        format!("cannot find local file '{}'", filename),
      )));
    }

    debug!("is remote but didn't find module");

    // 不是缓存/本地，尝试远程加载
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

`module_name`是一个字符串，表示需要加载的模块，它是根据要导入的文件的路径和包含“import”语句的文件的路径创建的。因此，一个通过`import "https://example.com/a.ts"`来加载的文件的内部包含了`import "./b.ts"`，将会被当做`import "https://example.com/b.ts"`。

`fetch_local_source`进入了文件系统，然后尝试基于文件名和文件类型来获取文件内容。

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
    // .mime文件可能不存在
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

当`fetch_remote_source`运行和下载远程的文件然后缓存在`$DENO_DIR/deps`时，他可能也会创建`.mime`文件：

```rust
fn fetch_remote_source(
    self: &Self,
    module_name: &str,
    filename: &str,
  ) -> DenoResult<Option<CodeFetchOutput>> {
    let p = Path::new(&filename);
    // 我们在`.deno/deps`目录中，缓存文件旁边的写入一个特殊的".mime"文件
    // 这个文件仅仅包含了媒体类型。
    let media_type_filename = [&filename, ".mime"].concat();
    let mt = Path::new(&media_type_filename);
    eprint!("Downloading {}...", &module_name); // no newline
    let maybe_source = http_util::fetch_sync_string(&module_name);
    if let Ok((source, content_type)) = maybe_source {
      eprintln!(""); // 下一行
      match p.parent() {
        Some(ref parent) => fs::create_dir_all(parent),
        None => Ok(()),
      }?;
      deno_fs::write_file(&p, &source, 0o666)?;
      // 删除可能存在的旧的.mime文件
      let _ = std::fs::remove_file(&media_type_filename);
      // 只在内容类型与扩展名不同时创建.mime文件
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

因此，解析逻辑如下所示：

* 如果`module_name`以一个远程url模式开始：
  * 如果带有`--reload`标志，强制下载文件和使用它。
  * 否则
    * 如果本地缓存文件存在，使用它。
    * 否则，下载文件到`$DENO_DIR/deps`目录，然后使用它。
* 如果`module_name`表示一个本地源文件，使用本地文件。





