# Callouts

## Introduction

A callout is typically an indication of a faulty hardware component in a system.
In OpenBMC, a callout is defined as any other error, via a YAML file. An example
would be `xyz.openbmc_project.Error.Callout.IIC`, to indicate an IIC callout.

The goal is to have applications post callouts using hardware terminology which
is familiar to them, such as a sysfs entry, or an IIC address. It would be up to
the OpenBMC error handling component to map such a callout to actual field
replaceable units (FRUs) on the system.

## Architecture and usage

An OpenBMC error has associated metadata, the same is true for a callout. Such
metadata would be defined in the callout YAML interface. Here is an example (for
xyz.openbmc_project.Error.Callout.IIC) :

```yaml
- name: IIC
  meta:
    - str: "CALLOUT_IIC_BUS=%s"
      type: string
    - str: "CALLOUT_IIC_ADDR=%hu"
      type: uint16
```

An application wanting to add an IIC callout will have to provide values for the
metadata fields above. These fields will also let the error handling component
figure out that this is in fact an IIC callout.

A callout is typically associated with an error log. For eg,
`xyz.openbmc_project.Error.Foo` may want to add an IIC callout. This is
indicated in Foo's YAML interface as follows :

```yaml
- name: Foo
  description: this is the error Foo
  inherits:
    - xyz.openbmc_project.Error.Callout.IIC
```

The way this inheritance will be implemented is that, Foo's metadata will
include Callout.IIC's as well, so an application wanting to add an IIC callout
will have to provide values for Foo and IIC metadata. Like mentioned before,
due to the presence of the Callout.IIC metadata, the error handling component
can figure out that the error Foo includes an IIC callout.

Currently, defined callout interfaces in turn inherit
`xyz.openbmc_project.Error.Callout.Device`, which has metadata common to
callouts :

```yaml
- name: Device
  meta:
    - str: "CALLOUT_ERRNO=%d"
      type: int32
    - str: "CALLOUT_DEVICE_PATH=%s"
      type: string
```

This way, say an application wants to express an IIC callout in terms of a
device path, for lack of IIC information. The application can add the callout
metadata fields for both Callout.Device and Callout.IIC, but provide values
only for Callout.Callout. That way the error handling component can still
decipher this as an IIC callout.

## Creation of a callout

This section talks about creation of a callout, once callout related metadata is
already in the journal.

Taking an example of a generic device callout here, but this would be the flow
in general :

- An application commits an error that has associated callout metadata. This
  will cause the error-log server to create a d-bus object for the error.

- The error-log server will detect that callout metadata is present, will
  extract the same and hand it over to a sub-module which will map callout
  metadata to one or more inventory object paths, and will create an
  association between the error object and the inventory object(s). The
  mapping from callout metadata to inventory objects is mostly done via
  the aid of code generated by the system MRW parsers.

- Generated code : consider a case where an application wants to callout
  an EEPROM on the BMC planar, via a device path, such as
  /sys/devices/platform/ahb/ahb:apb/1e78a000.i2c/i2c-11/i2c-11/11-0051/eeprom.
  This would have to be mapped to the BMC planar as the FRU to be called out.
  MRW parser(s) could be written which, for every device in the IIC subsystem,
  can provide a corresponding inventory object path. The error-log server, in
  this case, has to, by looking at the device path, determine that the device
  is on an IIC bus, and make use of the code generated to map the device to
  inventory objects.
  Similar MRW parsers could be written for other device subsystems.
