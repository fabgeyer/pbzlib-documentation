# pbzlib - Easy serialization of protobuf messages

This library is used for simplifying the serialization and deserialization of [protocol buffers](https://developers.google.com/protocol-buffers/) messages to/from files.
The main use-case is to save and read a large collection of protobuf messages of the same type, e.g. a dataset.
Each PBZ file file contains a header with the description of the protocol buffers, meaning that no compilation of `.proto` description file is required before reading the file.


## Official implementations

- [Python](https://github.com/fabgeyer/pbzlib-py) (reference implementation)
- [Go](https://github.com/fabgeyer/pbzlib-go)
- [Java](https://github.com/fabgeyer/pbzlib-java)
- [Rust](https://github.com/fabgeyer/pbzlib-rs)
- [C/C++](https://github.com/fabgeyer/pbzlib-c-cpp)

Additional contributions in other programming languages are welcome.


## Specification

PBZ files are binary data compressed using the [gzip](https://www.gzip.org) encoding.

The structure of a PBZ file is as follows:
- GZIP Header
- GZIP Data
    - **PBZ Magic** header (2 bytes): `[0x41, 0x42]`
    - **PBZ Message** encoded as Type-Length-Value
    - **PBZ Message** encoded as Type-Length-Value
    - **PBZ Message** encoded as Type-Length-Value
    - ...

Following the PBZ magic header, the following structure is used:
- The first PBZ message in a PBZ file is always a **protobuf file descriptor** encoded as PBZ Type-Length-Value.
- The next (optional) message is the protobuf version number as UTF8 string encoded as PBZ Type-Length-Value.

The rest of the PBZ messages contain the serialized protobuf messages.
To encode a series of protobuf messages of the same descriptor, the following structure is used:
- **Descriptor name of the next protobuf message(s)** as UTF8 string encoded as PBZ Type-Length-Value
- **Protobuf message** encoded as PBZ Type-Length-Value
- **Protobuf message** encoded as PBZ Type-Length-Value
- ...

### PBZ Type-Length-Value

All PBZ message are encoded using Type-Length-Value as follows:
- **PBZ Type** (1 byte) with one of the following values:
    - `T_FILE_DESCRIPTOR` (1)
    - `T_DESCRIPTOR_NAME` (2)
    - `T_MESSAGE` (3)
    - `T_PROTOBUF_VERSION` (4)
- **Length** of the following binary data encoded as unsigned [varint](https://developers.google.com/protocol-buffers/docs/encoding#varints), i.e. an unsigned integer encoded using one or more bytes,
- **Binary data** which is either a UTF8 string or a [serialized Protobuf object](https://developers.google.com/protocol-buffers/docs/encoding).

### Type `T_FILE_DESCRIPTOR`

This type contains the descriptors required for deserializating the protobuf messages contained in the PBZ file, i.e. the `.proto` files where the structure of the protobuf messages are defined.
It is a [FileDescriptorSet](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/descriptor.proto) message encoded using protobuf's serialization.

The encoded `.proto` files can be generated as follows:
```
$ protoc -I=. --include_imports --descriptor_set_out=messages.descr messages.proto
```
The content of the generated `messages.descr` can then directly be used for this PBZ object.

Alternatively, some protobuf libraries sometimes include this binary data in the compiled protobuf files of the target programming language.

This message type should only appears once in the PBZ file.

### Type `T_DESCRIPTOR_NAME`

This type contains the descriptor name of the next PBZ object(s) of type `T_MESSAGE` (e.g. `example.Message`).
It is encoded as UTF8 string.

This type is only required each time that the next protobuf message contained in the PBZ file has a different descriptor than the previous one.
In other words, if multiple protobuf messages have the same descriptor, the descriptor name is only required once.

This descriptor name needs to be defined in the aforementioned FileDescriptorSet, otherwise an error is raised.

### Type `T_MESSAGE`

This type contains an encoded protobuf message.
It is serialized using [Protobuf's serialization encoding](https://developers.google.com/protocol-buffers/docs/encoding), meaning that the serialization and deserialization functions are usually provided by the protobuf library of the target programming language.

### Type `T_PROTOBUF_VERSION`

This type contains the protobuf version number encoded as UTF8.
This type is optional and only used for assessing incompatibities between protobuf versions.
By default protobuf version 3 is assumed.

This message type should only appears once in the PBZ file.
