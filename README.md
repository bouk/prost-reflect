[![crates.io](https://img.shields.io/crates/v/prost-reflect.svg)](https://crates.io/crates/prost-reflect/)
[![docs.rs](https://docs.rs/prost-reflect/badge.svg)](https://docs.rs/prost-reflect/)
[![deps.rs](https://deps.rs/crate/prost-reflect/0.6.0/status.svg)](https://deps.rs/crate/prost-reflect)
![MSRV](https://img.shields.io/badge/rustc-1.54+-blue.svg)
[![Continuous integration](https://github.com/andrewhickman/prost-reflect/actions/workflows/ci.yml/badge.svg)](https://github.com/andrewhickman/prost-reflect/actions/workflows/ci.yml)
[![codecov.io](https://codecov.io/gh/andrewhickman/prost-reflect/branch/main/graph/badge.svg?token=E2OITYXO7M)](https://codecov.io/gh/andrewhickman/prost-reflect)
![Apache 2.0 OR MIT licensed](https://img.shields.io/badge/license-Apache2.0%2FMIT-blue.svg)

# prost-reflect

A protobuf library extending [`prost`](https://crates.io/crates/prost) with reflection support and dynamic messages.

## Usage

This crate provides support for dynamic protobuf messages. These are useful when the
protobuf type definition is not known ahead of time.

The main entry points into the API of this crate are:

- [`FileDescriptor`] wraps a [`FileDescriptorSet`][prost_types::FileDescriptorSet] output by 
  the protobuf compiler to provide an API for inspecting type definitions.
- [`DynamicMessage`] provides encoding, decoding and reflection of an arbitrary protobuf 
  message definition described by a [`MessageDescriptor`].

### Example - decoding

`DynamicMessage` does not implement [`Default`] since it needs a message descriptor to
function. To decode a protobuf byte stream into an instance of this type, use [`DynamicMessage::decode`]
to create a default value for the `MessageDescriptor` instance and merge into it:

```rust
use prost::Message;
use prost_types::FileDescriptorSet;
use prost_reflect::{DynamicMessage, FileDescriptor, Value};

let file_descriptor_set = FileDescriptorSet::decode(include_bytes!("file_descriptor_set.bin").as_ref()).unwrap();
let file_descriptor = FileDescriptor::new(file_descriptor_set).unwrap();
let message_descriptor = file_descriptor.get_message_by_name("package.MyMessage").unwrap();

let dynamic_message = DynamicMessage::decode(message_descriptor, b"\x08\x96\x01".as_ref()).unwrap();

assert_eq!(dynamic_message.get_field_by_name("foo").unwrap().as_ref(), &Value::I32(150));
```

### Example - JSON mapping

When the `serde` feature is enabled, `DynamicMessage` can be deserialized to and from the
[canonical JSON mapping](https://developers.google.com/protocol-buffers/docs/proto3#json)
defined for protobuf messages.

```rust
use prost::Message;
use prost_reflect::{DynamicMessage, FileDescriptor, Value};
use serde_json::de::Deserializer;

let file_descriptor = FileDescriptor::decode(include_bytes!("file_descriptor_set.bin").as_ref()).unwrap();
let message_descriptor = file_descriptor.get_message_by_name("package.MyMessage").unwrap();

let json = r#"{ "foo": 150 }"#;
let mut deserializer = Deserializer::from_str(json);
let dynamic_message = DynamicMessage::deserialize(message_descriptor, &mut deserializer).unwrap();
deserializer.end().unwrap();

assert_eq!(dynamic_message.get_field_by_name("foo").unwrap().as_ref(), &Value::I32(150));
```

### Example - implementing `ReflectMessage`

The [`ReflectMessage`] trait provides a `.descriptor()` method to get type information for a message. By default it is just implemented for `DynamicMessage`.

When the `reflect-well-known-types` feature is enabled, it is implemented for the well-known-types provided by [`prost-types`](https://docs.rs/prost-types/0.9.0/prost_types).

When the `derive` feature is enabled, it can be derived for [`Message`][prost::Message] implementations. The
derive macro takes the following parameters:

| Name            | Value |
|-----------------|-------|
| file_descriptor | An expression that resolves to a [`FileDescriptor`] containing the message type. The descriptor should be cached to avoid re-building it. |
| message_name    | The full name of the message, used to look it up within [`FileDescriptor`]. |

```rust
use prost::Message;
use prost_reflect::{FileDescriptor, ReflectMessage};
use once_cell::sync::Lazy;

static FILE_DESCRIPTOR: Lazy<FileDescriptor>
    = Lazy::new(|| FileDescriptor::decode(include_bytes!("file_descriptor_set.bin").as_ref()).unwrap());

#[derive(Message, ReflectMessage)]
#[prost_reflect(file_descriptor = "FILE_DESCRIPTOR", message_name = "package.MyMessage")]
pub struct MyMessage {}

let message = MyMessage {};
assert_eq!(message.descriptor().full_name(), "package.MyMessage");
```

If you are using `prost-build`, it can be configured to generate [`ReflectMessage`] implementations
for messages:

```rust,no_run
use prost_build::Config;

Config::new()
    .file_descriptor_set_path("file_descriptor_set.bin")
    .type_attribute(".package.MyMessage", "#[derive(::prost_reflect::ReflectMessage)]")
    .type_attribute(".package.MyMessage", "#[prost_reflect(file_descriptor = \"FILE_DESCRIPTOR\", message_name = \"package.MyMessage\")]")
    .compile_protos(&["src/package.proto"], &["src"])
    .unwrap();
```

## Minimum Supported Rust Version

Rust **1.54** or higher.

The minimum supported Rust version may be changed in the future, but it will be
done with a minor version bump.

## License

Licensed under either of

 * Apache License, Version 2.0
   ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license
   ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.

[`FileDescriptor`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/struct.FileDescriptor.html
[`DynamicMessage`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/struct.DynamicMessage.html
[`MessageDescriptor`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/struct.MessageDescriptor.html
[`MessageDescriptor`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/struct.MessageDescriptor.html
[`DynamicMessage::decode`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/struct.DynamicMessage.html#method.decode
[`ReflectMessage`]: https://docs.rs/prost-reflect/0.6.0/prost_reflect/trait.ReflectMessage.html

[`Default`]: https://doc.rust-lang.org/stable/core/default/trait.Default.html
[prost_types::FileDescriptorSet]: https://docs.rs/prost-types/latest/prost_types/struct.FileDescriptorSet.html
