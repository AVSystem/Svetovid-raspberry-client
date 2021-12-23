
# Anjay-raspberry-client [<img align="right" height="50px" src="https://avsystem.github.io/Anjay-doc/_images/avsystem_logo.png">](http://www.avsystem.com/)

## Overview

This the demonstration client for Linux based devices, specifically Raspberry Pi
running on Raspbian Buster. It presents a feature called FSDM (File System Data
Model) and implements a few LwM2M Objects directly on top of that.

Basic objects implemented are:

- Security (/0),
- Server (/1),
- Access Control (/2),
- Device (/3),
- Firmware Update (/5).

Additionally, we support [Raspberry Pi Sense Hat](https://www.raspberrypi.org/products/sense-hat/) extension board, and the following objects:

- Temperature (/3303),
- Accelerometer (/3313),
- Magnetometer (/3314),
- Barometer (/3315),
- Gyrometer (/3334),
- Addressable Text Display (/3341).

## What is FSDM (File System Data Model)

FSDM is a plugin for our Linux client that provides an easy way for LwM2M
Objects prototyping and development clasically done in C or C++. With the plugin
however, there is no such limitation, and objects can be implemented pretty much
in any language of choice. The only requirement is that the object structure
follows certain schema, and executables behave in the way expected by the LwM2M
Client that loads & manages them.

Currently, we have extensive support libraries and code-generators for Python
and sh, to get you started even faster.

## Installation instructions

Install `svetovid_21.12-raspberry_armhf.deb` on Raspberry Pi:

```sh
$ sudo dpkg -i svetovid_21.12-raspberry_armhf.deb
```

Note that this installs a `svetovid.service` systemd service, automatically
enabled, and starts-up the Client immediately. You may want to disable this
behavior in the following way:

```sh
$ sudo systemctl disable svetovid.service --now # disable and stop svetovid service
```

The basic installation package does not contain the FSDM plugin described above.
To install it, please run:

```sh
$ sudo dpkg -i svetovid-plugin-fsdm_21.12-raspberry_armhf.deb \
avsystem_svetovid-21.12-raspberry-Linux-fsdmtool-runtime-python.deb
```

If you have the [Raspberry Pi Sense Hat](https://www.raspberrypi.org/products/sense-hat/)
extension board, you may install a dedicated package to enable more objects:

```sh
$ sudo dpkg -i avsystem_svetovid-21.12-raspberry-Linux-sensehat.deb
```

## Configuration

Svetovid keeps configuration in JSON files. For Raspberry Pi, the default
location of these JSONs is ``/etc/svetovid/config``. Default configuration
directory may be overwritten by passing ``--conf-dir`` command line argument
when starting a Svetovid binary.

WARNING: Only following JSON files are supposed to be modified manually:

- ``security.json``
- ``server.json``
- ``svd.json``

Editing other JSON files in configuration directory may cause unexpected
behavior of the client.

NOTE: The LwM2M client process may modify these files at runtime. To avoid
your changes from being overwritten, make sure to stop the ``svetovid``
process before modifying the configuration (see Startup process and client
operation section).

### Global settings are stored in `svd.json` file

Note: This file can generally be left empty if you are fine with the defaults.

Example:
```json
{
    "device": {
        "endpoint_name": "urn:dev:os:0023C7-EXAMPLE_DEVICE",
        "udp_listen_port": 1234
    },
    "logging": {
        "default_log_level": "debug",
        "log_level": {
            "svd": "info"
        }
    },
    "in_buffer_size_b": 1024,
    "out_buffer_size_b": 1024,
    "msg_cache_size_b": 65536
}
```

The following configuration options are recognized:

- `device.endpoint_name` - if set, the LwM2M Endpoint Client Name will be
  literally set to the configured value. Otherwise, it will be set to
  `urn:dev:os:B827EB-<SERIAL_NUMBER>`, with `<SERIAL_NUMBER>` replaced by the
  actual serial number of the Pi - the value that can be read in `/proc/cpuinfo`
  and `/sys/firmware/devicetree/base/serial-number`.

- `device.udp_listen_port` - force binding to a specific UDP port. If set to a
  non-zero value, all UDP sockets created by the LwM2M client will be bound to
  configured port. Otherwise, random ephemeral ports will be used.

- `device.server_initiated_bootstrap` - enables / disables LwM2M Server
  Initiated bootstrap support. If set to true, connection to the Bootstrap
  Server will be closed immediately after making a successful connection to
  any regular LwM2M Server and only opened again if (re)connection to a regular
  server is rejected. Default value is: `0`.

- ``logging.default_log_level`` - log level applied to messages in case no
  more specific log level exists.

  Acceptable values:
  - "trace" (log all messages)
  - "debug"
  - "info"
  - "warning"
  - "error"
  - "quiet" (do not log anything)

  Default value: "info"

- ``logging.log_level.MODULE_NAME`` - log level applied to messages originating
  from ``MODULE_NAME`` only. Can be used to selectively control logging level.

- `in_buffer_size_b` - size (in bytes) of the buffer used for storing incoming
  LwM2M messages. The client will not be able to handle packets bigger than this
  size.

  Default value: 4096

- `out_buffer_size_b` - size (in bytes) of the buffer used for storing outgoing
  LwM2M messages. In cases where the message sent would exceed this size, the
  client will attempt a BLOCK-wise CoAP transfer instead.

  Default value: 4096

- ``msg_cache_size_b`` - size (in bytes) of the buffer used for storing outgoing
  LwM2M messages. When the client receives a duplicate request while an
  already-prepared response is in the cache, it is used instead of generating a
  new one. Cached messages are removed after their validity expires. If total
  size of cached messages exceeds configured value, oldest entries are evicted
  to make room for fresh ones.

  Setting this value to 0 disables message caching. In such case, the client
  will handle all received retransmitted requests as if they were new ones,
  which may result in performing non-idempotent operations multiple times.

  Default value: 65536

- ``retry_after_s`` - enables / disables reconnection policy which after
  specified period of time (in seconds) after all server connections failed
  performs a reconnection attempt. Value of 0 disables reconnection attempts,
  and causes the client to shutdown if it is unable to establish any connection.

  Default value: 30

- ``dirs.persistence`` - path to the persistence directory. That path MUST NOT
  get cleared on FW update.

  Default value: `SVETOVID_PERSISTENCE_DIR` set during compile time.

- ``dirs.volatile_persistence`` - path to a volatile persistence directory. That
  path MUST be cleared on FW update, but persist across reboots.

  Default value: `/etc/svetovid/persistence`

- ``dirs.plugins`` - path to svd plugin installation directory.

  Default value: `/usr/lib/svetovid`

- ``dirs.temp`` - path to a directory used for temporary file storage.

  Default value: `/tmp`

- ``dirs.firmware_download_dir`` - path where PULL FW downloads will be kept.
  It MAY be cleared on FW update, but SHOULD persist across reboots in order to
  support firmware download resumption.

  Default value: `/tmp`


### Server connection settings are stored in `security.json` and `server.json`

The default configuration is designed to let you easily connect to our
[Coiote IoT Device Management](https://www.avsystem.com/products/coiote-iot-device-management-platform/)
LwM2M Server platform. Please register at https://www.avsystem.com/try-anjay/ to
get access.

In the ``security.json`` file you're gonna need to change the
`privkey_or_psk_hex` with hexlified pre-shared-key of your choice. To convert
raw string to hexlified string, you can use:

```sh
$ echo -n 'your-secret-key' | xxd -p
```

You can now restart or start (if not started already) the LwM2M Client:
```sh
# if you disabled svetovid.service in previous steps
$ svetovid
# or if you intend to use systemd to manage svetovid process
$ sudo systemctl restart svetovid.service
```

#### Complete reference for the `security.json` file options

- `server_uri` - LwM2M Server URI ("coap://" or "coaps://" URI, depending on the
  `security_mode` value),

- `is_bootstrap` - Bootstrap Server (boolean)

- `security_mode` - Security Mode (one of: "psk", "nosec", "cert")

- `pubkey_or_identity_hex` - Public Key or Identity (hex string). NOTE: this
  **must** be a hex string, even if the value is in fact a printable text. For
  example, if the PSK identity is supposed to be "identity", this value should
  be set to "6964656e74697479".

- `server_pubkey_hex` - Server Public Key (hex string; see NOTE above)

- `privkey_or_psk_hex` - Secret Key (hex string; see NOTE above)

- `ssid` - Short Server ID (1-65534)

- `holdoff_s` - Client Hold Off Time (seconds)

- `bs_timeout_s` - Bootstrap-Server Account Timeout (seconds)

#### Complete reference for the `server.json` file options

- `ssid` - Short Server ID (1-65534, must match `ssid` of some Security Object
  Instance)

- `lifetime` - Lifetime (seconds)

- `default_min_period` - Default Minimum Period (seconds)

- `default_max_period` - Default Maximum Period (seconds)

- `binding` - Binding (one of: "U", "UQ")

- `notification_storing` - Notification Storing When Disabled or Offline
  (boolean)

- `disable_timeout` - Disable Timeout (seconds)

#### Using a Bootstrap Server

When using a Bootstrap Server, it may modify the contents of the Security and
Server objects. These changes will **NOT** be written back to `security.json` or
`server.json` files - instead, they will be persisted into
`/etc/svetovid/persistence/persistence.dat`. Note that this is a binary file
that is not intended for user modification.

The configuration in `security.json` and `server.json` will take preference if
the `persistence.dat` file doesn't exist, or if either of the JSON files is
newer than the last time Svetovid has been bootstrapped from them. In other
words, if you modify or `touch` the JSON files, they shall take preference.


## Developing custom objects

FSDM comes with a helper tool for generating stubs of all required scripts.
Run `svetovid-fsdmtool --help` to see up-to-date help message with usage examples.

### First, how does the FSDM actually work?

#### Structure

The FSDM plugin maps specific directory (`/etc/svetovid/dm` by default) and its
structure to LwM2M Objects, Instances and Resources. The recognized structure is
as follows:

- `/etc/svetovid/dm/`
    - `object_id/` (e.g. `3333`) - directory representing an LwM2M Object with
      given ID.
        - `resources/` - directory containing scripts used to access individual
          Resources. Names of individual Resources MUST exactly correspond to
          their Resource IDs. (autogenerated by `svetovid-fsdmtool`),
        - `instances` - an optional executable script for managing instances
          (autogenerated by `svetovid-fsdmtool`),
        - `transaction` - an optional executable script used to handle
          transactional processing of object resources (this is a complicated
          topic, not yet covered in this demo).

NOTE: The `svetovid-fsdmtool` script, when generating object's structure, also
adds human-readable symlinks under `object_id/` directory to resources located
under `object_id/resources/`.

Every LwM2M operation is mapped to execution of one or more scripts located
under the `object_id/`. Examples:

- LwM2M Read on some `/Object ID/Instance ID/Resource ID` will be transformed
  into:
    - getting the list of instances from `Object ID/instances` script to verify
      if the targeted instance exists,
    - calling the `Object ID/Resource ID` script to read the value of the
      resource for that instance (the Instance ID is passed to `Resource ID`
      script as parameter).
- LwM2M Read on some `/Object ID/Instance ID` will be transformed into:
    - getting the list of instances from `Object ID/instances` script to
      enumerate instance to be read,
    - calling resource scripts (as above), but for every present instance.
- LwM2M Observe is transformed into:
    - if the "external notify" mechanism is disabled for a given LwM2M Object
      (default): periodical LwM2M Reads to see if resource values changed. You
      can control the frequency of reads/notifications by `pmin` and `pmax`
      LwM2M Attributes.
    - it is the user responsibility to notify Svetovid about the following
      changes in FSDM:
      - list of valid Instance IDs for an object has changed for **other**
        reason than receiving the Create or Delete operation calls from Svetovid
        itself
      - readable resource has changed its value for **other** reason than
        receiving the Write, Reset or Clear operation calls from Svetovid itself
- LwM2M Delete is transformed into:
    - calls to `instances` script to delete instances.

#### Input and output

Resource scripts obtain necessary information either from parameters passed to
them or from the standard input. For example, the LwM2M Write on a Resource
containing payload "example", will execute the corresponding Resource script,
passing the "example" string on its standard input.

Resource scripts return values to Svetovid via standard output when they're used
to extract the value they represent. Apart from that, the scripts' exit codes
are translated to CoAP error responses. In the Python implementations, any
errors are communicated by exceptions, which are in turn translated to error
codes by the runtime in a way transparent to the user.

LwM2M Execute arguments are passed as arguments to the script body. In Python
implementations, execute arguments are passed as a parameter to `execute()`
method.

### Example

Say you want to implement the
[Time Object](http://www.openmobilealliance.org/tech/profiles/lwm2m/3333.xml)
(/3333). It has a few basic read/write resources. You can start with generating
the stub:

```sh
$ sudo svetovid-fsdmtool generate --object 3333 --output-dir /etc/svetovid/dm --generator python
```

This creates `/etc/svetovid/dm/3333` directory containing (note the directory
has the structure as described above):

```
├── Application_Type -> resources/5750
├── Current_Time -> resources/5506
├── Fractional_Time -> resources/5507
├── instances
└── resources
    ├── 5506
    ├── 5507
    └── 5750
```

Let's start with an "Application Type" resource implementation. The placeholders
for `read`, `write`, and `reset` need to be filled in with some actual logic.

The first problem to think about is: how do we store the incoming Application
Type written by the Server? Svetovid supports simple key-value store which is
accessible from Python scripts via `KvStore` class. It provides a very simple interface:

- `KvStore(namespace)` - constructor that takes `namespace` parameter as an
  argument. This `namespace` should be set to e.g. an Object ID in which the
  `KvStore` is used. The idea is that when many objects are implemented and
  utilize the store, there's a risk of key-collision between them. The 
  `namespace` parameter is supposed to uniquely distinguish between different 
  objects.
- `get(key, default=None)` - for getting the given `key`, or returning the
  `default` value if it is not present in the store
- `set(key, value)` - for creating/replacing the `value` assigned to a given
  `key`,
- `delete(key)` - for deleting the `key` and the value associated with it.

We can use this to implement `Application Type` resource as follows:

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-

from fsdm import ResourceHandler, CoapError, DataType, KvStore

import sys # for sys.stdout.write() and sys.stdin.read()

class ResourceHandler_3333_5750(ResourceHandler):
    NAME = "Application Type"
    DESCRIPTION = '''\
The application type of the sensor or actuator as a string depending
 * on the use case.'''
    DATATYPE = DataType.STRING
    EXTERNAL_NOTIFY = False

    def read(self,
             instance_id,            # int
             resource_instance_id):  # int for multiple resources, None otherwise
        value = KvStore(namespace=3333).get('application_type')
        if value is None:
            # The value was not set, so it's not found.
            raise CoapError.NOT_FOUND

        # The value is present within the store, thus we can print it on stdout.
        # The important thing here is to remember to return string-typed resources
        # with sys.stdout.write(), as print() adds unnecessary newline character, so
        # if we used it instead, the value presented to the server would contain that
        # trailing newline character.
        sys.stdout.write(value)


    def write(self,
              instance_id,            # int
              resource_instance_id):  # int for multiple resources, None otherwise
        # All we need to do is to assign a value to the application_type key.
        KvStore(namespace=3333).set('application_type', sys.stdin.read())


    def reset(self,
              instance_id):  # int
        # We reset the resource to its original state by simply deleting the application_type
        # key
        KvStore(namespace=3333).delete('application_type')



if __name__ == '__main__':
    ResourceHandler_3333_5750().main()

```

Implementation of other resources is even simpler (assuming we make them read-only). For example,
the `Fractional Time` resource can be implemented as follows:

```python
#!/usr/bin/env python
# -*- encoding: utf-8 -*-

from fsdm import ResourceHandler, CoapError, DataType, KvStore


class ResourceHandler_3333_5506(ResourceHandler):
    NAME = "Current Time"
    DESCRIPTION = '''\
Unix Time. A signed integer representing the number of seconds since
 * Jan 1st, 1970 in the UTC time zone.'''
    DATATYPE = DataType.TIME
    EXTERNAL_NOTIFY = False

    def read(self,
             instance_id,            # int
             resource_instance_id):  # int for multiple resources, None otherwise
        # It's just that simple!
        import time
        print(int(time.time()))


    def write(self,
              instance_id,            # int
              resource_instance_id):  # int for multiple resources, None otherwise
        # NOTE: Implement this if you want to be able to change time on your system.
        raise CoapError.NOT_IMPLEMENTED

    def reset(self,
              instance_id):  # int
        # NOTE: reset resource to its original state. You can either set it to
        # a default value or delete the resource.
        pass



if __name__ == '__main__':
    ResourceHandler_3333_5506().main()
```

For more complex examples install
`avsystem_svetovid-21.12-raspberry-Linux-sensehat.deb` package as
described above, and have a look at other objects impementations in
`/etc/svetovid/dm`.

NOTE: If you create FSDM scripts for an object ID that is already implemented in
the core client, the FSDM implementation will take precedence. Please note that
this might not be the case in case of other plugins (like the Sense Hat one), as
the plugin loading order will decide - so you may prefer not to install the
Sense Hat plugin if you intend to implement these objects yourself.

### External notify mechanism

To activate "external notify" mechanism for an object instance or a resource,
you need to explicitly enable that mode in the resource or instances scripts:

For Python scripts generated by `fsdmtool`, for readable entities, the class
constant `EXTERNAL_NOTIFY` should be set to `True` (default value is `False`).

After enabling the functionality, it is possible to use special Unix domain
socket to notify about value changes. The socket is created after Svetovid
launch in Svetovid temporary directory (by default: `/tmp/fsdm_local_socket`).

You may send JSON containing information about changed state of instances and
resources as follows:

```json
{ "notify": ["/10", "/20", "/9/0/1", "/9/0/2"] }
```

In example above we want to inform that:

- instances lists of objects 10 and 20 have changed
- values of resources ``/9/0/1`` and ``/9/0/2`` have changed

If any of these resources is indeed observed, Svetovid will then invoke the Read
operation on the appropriate FSDM script to query the actual resource value.

To send the message through the socket, you can use standard tools like `nc` or
`socat`.

#### nc

```sh
echo '{ "notify": ["/10", "/20", "/9/0/1", "/9/0/2"] }' | nc -NU /tmp/fsdm_local_socket
```

NOTE: Option `-N` is set because `nc` should shutdown the socket after EOF on
the input.

#### socat

```sh
echo '{ "notify": ["/10", "/20", "/9/0/1", "/9/0/2"] }' | socat - UNIX-CONNECT:/tmp/fsdm_local_socket
```

#### Native socket API

You can use standard socket API of your preferred programming language. These
are important things to remember about user-side socket:

- socket domain should be `AF_UNIX` / `AF_LOCAL`
- socket type should be `SOCK_STREAM`
- after sending whole message the socket should be shut down for further
  transmissions (`SHUT_WR` flag)

#### Response

As a result of triggering a notify a response is sent through the socket. There
are 3 kinds of result:

1. `{"result": "OK"}`: There were no errors during triggering a notify.

2. `{"result": "warning", "details": [ ... ] }`: In this case some of entries
   could not be processed and the reasons for each entry are indicated in
   `details` section. Entries omitted in `details` section were perfectly
   valid and there is no need to try notifying them again.

3. `{"result": "error", "details": "..." }`: There was some serious
   problem with execution of user request (e.g. parsing error). In this
   case **all entries** should be considered as **not processed**.

#### Examples

`"details"` section for `"OK"` result is absent:

```js
user@host $ echo '{ "notify": ["/1337"] }' | nc -NU /tmp/fsdm_local_socket
{
    "result": "OK"
}
user@host $
```

`"details"` for `"warning"` result is an array of failure reasons:

```js
user@host $ echo '{ "notify": [":-)", "/1/2/3"] }' | nc -NU /tmp/fsdm_local_socket
{
    "result": "warning",
    "details": [
        {
            "path": ":-)",
            "reason": "not object or resource path"
        },
        {
            "path": "\/1",
            "reason": "non-FSDM object"
        }
    ]
}
user@host $
```

`"details"` for `"error"` result is a single diagnostic string:

```js
user@host $ echo abcdefgh | nc -NU /tmp/fsdm_local_socket
{
    "result": "error",
    "details": "malformed input"
}
user@host $
```
