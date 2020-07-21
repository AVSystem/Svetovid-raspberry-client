# Svetovid-raspberry-client [<img align="right" height="50px" src="https://avsystem.github.io/Anjay-doc/_images/avsystem_logo.png">](http://www.avsystem.com/)

## Overview

This the demonstration client for Linux based devices, specifically Raspberry Pi running on Raspbian Buster. It presents a feature called FSDM (File System Data Model) and implements a few LwM2M Objects directly on top of that.

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

FSDM is a plugin for our Linux client that provides an easy way for LwM2M Objects prototyping and development clasically done in C or C++. With the plugin however, there is no such limitation, and objects can be implemented pretty much in any language of choice. The only requirement is that the object structure follows certain schema, and executables behave in the way expected by the LwM2M Client that loads & manages them.

Currently, we have extensive support libraries and code-generators for Python and sh, to get you started even faster.

## Installation instructions

Install `svetovid_20.07_armhf.deb` on Raspberry Pi:

```sh
$ sudo dpkg -i svetovid_20.07_armhf.deb
```

Note that this installs a `svetovid.service` systemd service, automatically enabled, and starts-up the Client immediately. You may want to disable this behavior in the following way:

```sh
$ sudo systemctl disable svetovid.service --now # disable and stop svetovid service
```

If you have the [Raspberry Pi Sense Hat](https://www.raspberrypi.org/products/sense-hat/) extension board, you may install additional packages to enable more objects:

```sh
$ sudo dpkg -i svetovid-plugin-fsdm_20.07_armhf.deb \
avsystem_svetovid-20.07-Linux-fsdmtool-runtime-python.deb \
avsystem_svetovid-20.07-Linux-sensehat.deb
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

### Server connection settings are stored in `security.json` and `server.json`

The default configuration is designed to let you easily connect to our [Coiote IoT Device
Management](https://www.avsystem.com/products/coiote-iot-device-management-platform/) LwM2M Server platform. Please register at https://www.avsystem.com/try-anjay/ to get access.

In the ``security.json`` file you're gonna need to change the `privkey_or_psk_hex` with hexlified pre-shared-key of your choice. To convert raw string to hexlified string, you can use:

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

### Settings not related to LwM2M server connections are stored in `svd.json` file

Example:
```json
{
    "logging": {
        "default_log_level": "debug",
    }
}
```

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


## Developing custom objects

FSDM comes with a helper tool for generating stubs of all required scripts.
Run `svetovid-fsdmtool --help` to see up-to-date help message with usage examples.

### First, how does the FSDM actually work?

#### Structure

The FSDM plugin maps specific directory (`/etc/svetovid/dm` by default) and its structure to LwM2M Objects, Instances and Resources. The recognized structure is as follows:

- `/etc/svetovid/dm/`
    - `object_id/` (e.g. `3333`) - directory representing an LwM2M Object with given ID.
        - `resources/` - directory containing scripts used to access individual Resources. Names of individual Resources MUST exactly correspond to their Resource IDs. (autogenerated by `svetovid-fsdmtool`),
        - `instances` - an optional executable script for managing instances (autogenerated by `svetovid-fsdmtool`),
        - `transaction` - an optional executable script used to handle transactional processing of object resources (this is a complicated topic, not yet covered in this demo).

NOTE: The `svetovid-fsdmtool` script, when generating object's structure, also adds human-readable symlinks under `object_id/` directory to resources located under `object_id/resources/`.

Every LwM2M operation is mapped to execution of one or more scripts located under the `object_id/`. Examples:

- LwM2M Read on some `/Object ID/Instance ID/Resource ID` will be transformed into:
    - getting the list of instances from `Object ID/instances` script to verify if the targeted instance exists,
    - calling the `Object ID/Resource ID` script to read the value of the resource for that instance (the Instance ID is passed to `Resource ID` script as parameter).
- LwM2M Read on some `/Object ID/Instance ID` will be transformed into:
    - getting the list of instances from `Object ID/instances` script to enumerate instance to
    be read,
    - calling resource scripts (as above), but for every present instance.
- LwM2M Observe is transformed into:
    - periodical LwM2M Reads to see if resource values changed. You can control the frequency of reads/notifications by `pmin` and `pmax` LwM2M Attributes.
- LwM2M Delete is transformed into:
    - calls to `instances` script to delete instances.

#### Input and output

Resource scripts obtain necessary information either from parameters passed to a them or from the standard input. For example, the LwM2M Write on a Resource containing payload "example", will execute the corresponding Resource script, passing the "example" string on its standard input.

Resource scripts return values to Svetovid via standard output when they're used to extract the value they represent. Apart from that, the scripts' exit codes are translated to CoAP error responses. In the Python implementations, any errors are communicated by exceptions, which are in turn translated to error codes by the runtime in a way transparent to the user.

LwM2M Execute arguments are passed as arguments to the script body. In Python implementations, execute arguments are passed as a parameter to `execute()` method.

### Example

Say you want to implement the [Time Object](http://www.openmobilealliance.org/tech/profiles/lwm2m/3333.xml) (/3333). It has a few basic read/write resources. You can start with generating the stub:

```sh
$ sudo svetovid-fsdmtool generate --object 3333 --output-dir /etc/svetovid/dm --generator python
```

This creates `/etc/svetovid/dm/3333` directory containing (note the directory has the structure as described above):

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

Let's start with an "Application Type" resource implementation. The placeholders for `read`, `write`, and `reset` need to be filled in with some actual logic.

The first problem to think about is: how do we store the incoming Application Type written by the Server? Svetovid supports simple key-value store which is accessible from Python scripts via `KvStore` class. It provides a very simple interface:

- `KvStore(namespace)` - constructor that takes `namespace` parameter as an argument. This `namespace` should be set to e.g. an Object ID in which the `KvStore` is used. The idea is that when many objects are implemented and utilize the store, there's a risk of key-collision between them. The `namespace` parameter is supposed to uniquely distinguish between different objects.
- `get(key, default=None)` - for getting the given `key`, or returning the `default` value if it is not present in the store
- `set(key, value)` - for creating/replacing the `value` assigned to a given `key`,
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

For more complex examples install `avsystem_svetovid-20.07-Linux-sensehat.deb` package as described above, and have a look at other objects impementations in `/etc/svetovid/dm`.