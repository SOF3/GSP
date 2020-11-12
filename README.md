# GspPluginLoader

A Generalized Subprocess Plugin (GSP) is a PocketMine plugin written in any language supporting standard IO.
The GSP is initialized as a standalone subprocess of the PocketMine process,
and interact with the PocketMine API using the standard IO interface.

## Plugin format
GSP is a directory in the plugins directory with a `gsp.yml` file.
It shares the same format as the usual `plugin.yml`,
except in the `main` attribute, instead of a PHP class string,
a command line shall be provided,
which is used to spawn the GSP as a subprocess.

The command line can be a string or an array.
See the documentation of [`proc_open`](https://php.net/proc-open) for details.
It can also be a mapping of strings or arrays,
in which the key should be values of `PHP_OS` to specify the command line based on operating system.
If more configuration is required, the developer can create a batch/powershell/shell script
and invoke it from the command line.

The command line is executed with cwd set to the parent directory of the `gsp.yml`.
The environment variable `INVOKED_FROM_GSP_RUNTIME` is set to `1`.

## Protocol
The subprocess communicates with the parent PHP process
using standard IO in a binary protocol as specified below.

### Messages
PocketMine sends "messages" to the subprocess through the stdin,
and the subprocess sends "messages" to PocketMine through the stdout.
GspPluginLoader also forwards the stderr output to PocketMine's main logger at DEBUG level asynchronously.

Each "message" starts with an unsigned 16-bit integer, indicating the message type.

#### Startup messages
##### Shebang message
In case the first two bytes are `#!` (in ASCII encoding),
the stream is skipped until a Line Feed byte is encountered.

The shebang message is only allowed as the first message in the stream.
Further occurrences will crash the plugin.

##### Handshake message
```
00 01 "P" "M" "G" "S" "P" "S" VERSION ENABLER DISABLER
```

- `VERSION` is a 16-bit integer indicating the supported GSP protocol version.
The current version is `00 01`.
- `ENABLER` is the GCBID of the callback to invoke when the plugin shall be enabled.
- `DISABLER` is the GCBID of the callback to invoke when the plugin shall be enabled.

If the GSP protocol version is mismatched,
GspPluginLoader sends a `SIGTERM` signal to the subprocess.
Therefore, the GSP should not initialize anything other than basic language runtime setup
before sending the handshake message.

#### GOA
All PHP objects accessible by subprocesses are stored in the GSP Object Arena (GOA).
Each object is identified by a GOAID,
which is a 64-bit signed integer assigned by GspPluginLoader.
Although the current implementation uses the `spl_object_id` as the GOAID,
this is subject to change in the future without affecting protocol version.

An object is added to the GOA when its ID is passed in any message to the subprocess,
and removed when the ObjectDeallocate message is sent from the subprocess.
The GOA is not automatically garbage-collected,
so it is the responsibility of the subprocess to send the ObjectDeallocate message frequently.

It is guaranteed that the same PHP object always uses the same GOAID,
but it is not guaranteed that the same GOAID implies the same PHP object
if the GOAID was Objectdeallocated.

##### ObjectDeallocate message
```
01 01
```

#### Callbacks
Callbacks are asynchronous invocations initiated by GspPluginLoader.
Each callback is allocated with a GCBID (totally orthogonal to GOAID),
which is a 64-bit signed integer assigned by the subprocess.
An assigned GCBID must be positive;
a negative value is never allowed,
and a zero value implies no-op (returns null).

A callback must be invokable as long as its GCBID is passed to GspPluginLoader,
until it is deallocated.
Callbacks are deallocated when the CallbackDeallocate message is sent from GspPluginLoader,
which happens when PHP garbage-collects the wrapper object for the callback.

##### CallbackDeallocate message
```
02 01 GCBID
```

- `GCBID` is the callback to deallocate.

Sent from GspPluginLoader,
allows deallocation of the callback.

##### CallbackToClosure message
```
02 02 GCBID
```

- `GCBID` is the callback to convert to closure.

Sent from subprocess,
requests to create a PHP Closure object that invokes the callback.
The closure would block until the callback completes.

##### CallbackInvoke message
```
02 03 INVOKE_ID GCBID ARG_COUNT ( PARAM )*
```

- `INVOKE_ID` is the invocation ID to pass when the callback completes
- `GCBID` is the callback to invoke.
- `ARG_COUNT` is the 64-bit signed integer representing the number of arguments to pass.
- `PARAM` are the PHP values to pass as arguments.

Sent from GspPluginLoader,
invokes the callback with the passed arguments.

The callback must return or throw by sending the CallbackComplete message.

##### CallbackComplete message
```
02 04 INVOKE_ID ERROR VALUE
```

- `INVOKE_ID` is the `INVOKE_ID` passed in the CallbackInvoke message.
- `ERROR` is a 8-bit byte indicating whether the callback should throw (`01`) or return (`00`)
- `VALUE` is the PHP value to return/throw

#### Accessing PHP
##### FunctionCall message
```
03 00 RETURN THROW FUNCTION_NAME ARG_COUNT ( PARAM )*
```

- `RETURN` is the GCBID of the callback to invoke with the function return value.
- `THROW` is the GCBID of the callback to invoke with the function return value.
- `FUNCTION_NAME` is the name of the function to invoke.
- `ARG_COUNT` is the 64-bit signed integer representing the number of arguments to pass.
- `PARAM` are the PHP values to pass as arguments.

##### MethodCall message
```
03 01 RETURN THROW GOAID METHOD_NAME ARG_COUNT ( PARAM )*
```

- `RETURN` is the GCBID of the callback to invoke with the method return value.
- `THROW` is the GCBID of the callback to invoke with the method return value.
- `GOAID` is the object to call the method on.
- `METHOD_NAME` is the name of the function to invoke.
- `ARG_COUNT` is the 64-bit signed integer representing the number of arguments to pass.
- `PARAM` are the PHP values to pass as arguments.

##### PropertyGet message
```
03 02 GCBID GOAID PROPERTY_NAME
```

- `GCBID` is the callback to invoke with the property value.
- `GOAID` is the object to get the property on.
- `PROPERTY_NAME` is the name of the property.

##### PropertySet message
```
03 03 GCBID GOAID PROPERTY_NAME VALUE
```

- `GCBID` is the callback to invoke with the original value of the property.
- `GOAID` is the object to get the property on.
- `PROPERTY_NAME` is the name of the property.

#### Concurrency
##### Batch message
```
04 00 ( MESSAGE )*
```

- Each `MESSAGE` encodes a message to dispatch.

This guarantees that the messages are dispatched on the main thread consecutively.
This has no effect on messages that are not dispatched on the main thread.

##### BlockStart message
```
04 01 GCBID
```

- `GCBID` is the callback to invoke when the lock starts

Blocks the main thread to only process messages from the subprocess until LockEnd is received.

##### BlockEnd message
```
04 02
```

Unblocks the main thread.

### PHP values
A PHP value is encoded on the protocol by a discriminant byte followed by the data.

#### Null
```
00
```

#### Boolean
```
01 VALUE
```

- `VALUE` is `01` for true, `00` for false.

#### Integer
```
02 VALUE
```

- `VALUE` is the little-endian signed 64-bit encoding of the integer.

Note that PocketMine only supports 64-bit systems,
and all integers in PHP are signed.

#### Float
```
03 VALUE
```

- `VALUE` is the little-endian double-precision IEEE-754 encoding of the float.

#### String
```
04 LENGTH BUFFER
```

- `LENGTH` is a little-endian signed 64-bit encoding of the length of the string.
- `BUFFER` is a byte array of `LENGTH` bytes, representing the contents of the string.

Note that PHP strings are binary byte arrays and encoding-insensitive.

#### Array
```
05 GOAID
```

- `GOAID` is the ArrayAdapter object allocated for the array.

To access array contents,
invoke the appropriate methods on the ArrayAdapter object.

#### Object
```
06 ID
```

- `ID` is the GOAID of the object.

See the "GOA" section above.
