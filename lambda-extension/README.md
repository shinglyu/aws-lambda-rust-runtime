# Runtime Extensions for AWS Lambda in Rust

[![Docs](https://docs.rs/lambda_extension/badge.svg)](https://docs.rs/lambda_extension)

**`lambda-extension`** is a library that makes it easy to write [AWS Lambda Runtime Extensions](https://docs.aws.amazon.com/lambda/latest/dg/using-extensions.html) in Rust. It also helps with using [Lambda Logs API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-logs-api.html).

## Example extensions

### Simple extension

The code below creates a simple extension that's registered to every `INVOKE` and `SHUTDOWN` events, and logs them in CloudWatch.

```rust,no_run
use lambda_extension::{service_fn, Error, LambdaEvent, NextEvent};

async fn my_extension(event: LambdaEvent) -> Result<(), Error> {
    match event.next {
        NextEvent::Shutdown(_e) => {
            // do something with the shutdown event
        }
        NextEvent::Invoke(_e) => {
            // do something with the invoke event
        }
    }
    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    tracing_subscriber::fmt()
        .with_max_level(tracing::Level::INFO)
        .with_ansi(false)
        .without_time()
        .init();

    let func = service_fn(my_extension);
    lambda_extension::run(func).await
}

```

### Log processor extension

```rust,no_run
use lambda_extension::{service_fn, Error, Extension, LambdaLog, LambdaLogRecord, SharedService};
use tracing::info;

async fn handler(logs: Vec<LambdaLog>) -> Result<(), Error> {
    for log in logs {
        match log.record {
            LambdaLogRecord::Function(_record) => {
                // do something with the function log record
            },
            LambdaLogRecord::Extension(_record) => {
                // do something with the extension log record
            },
            },
            _ => (),
        }
    }

    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let logs_processor = SharedService::new(service_fn(handler));

    Extension::new().with_logs_processor(logs_processor).run().await?;

    Ok(())
}

```

## Deployment

Lambda extensions can be added to your functions either using [Lambda layers](https://docs.aws.amazon.com/lambda/latest/dg/using-extensions.html#using-extensions-config), or adding them to [containers images](https://docs.aws.amazon.com/lambda/latest/dg/using-extensions.html#invocation-extensions-images).

Regardless of how you deploy them, the extensions MUST be compiled against the same architecture that your lambda functions runs on.

### Building extensions

Once you've decided which target you'll use, you can install it by running the next `rustup` command:

```bash
$ rustup target add x86_64-unknown-linux-musl
```

Then, you can compile the extension against that target:

```bash
$ cargo build -p lambda_extension --example basic --release --target x86_64-unknown-linux-musl
```

This previous command will generate a binary file in `target/x86_64-unknown-linux-musl/release/examples` called `basic`. When the extension is registered with the [Runtime Extensions API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html#runtimes-extensions-api-reg), that's the name that the extension will be registered with. If you want to register the extension with a different name, you only have to rename this binary file and deploy it with the new name.