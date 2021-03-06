---
layout: legacy
title: Provider Development
---

Provider Development
====================

Information about writing providers to provide implementation for types.

* * *

About
-----

The core of Puppet's cross-platform support is via Resource
Providers, which are essentially back-ends that implement support
for a specific implementation of a given resource type. For
instance, there are more than 20 package providers, including
providers for package formats like dpkg and rpm along with
high-level package managers like apt and yum. A provider's main job
is to wrap client-side tools, usually by just calling out to those
tools with the right information.

Not all resource types have or need providers, but any resource
type concerned about portability will likely need them.

We will use the apt and dpkg package providers as examples
throughout this document, and the examples used are current as of
0.23.0.

## Declaration

Providers are always associated with a single resource type, so
they are created by calling the provide class method on that
resource type. When declarating a provider, you can specify a
parent class --- for instance, all package providers have a common
parent class:

    Puppet::Type.type(:package).provide :dpkg, :parent => Puppet::Provider::Package do
        desc "..."
        ...
    end

Note the call desc there; it sets the documentation for this
provider, and should include everything necessary for someone to
use this provider.

Providers can also specify another provider (from the same resource
type) as their parent:

    Puppet::Type.type(:package).provide :apt, :parent => :dpkg, :source => :dpkg do
        ...
    end

Note that we're also specifying that this provider uses the dpkg
source; this tells Puppet to deduplicate packages from dpkg and
apt, so the same package does not show up in an instance list from
each provider type. Puppet defaults to creating a new source for
each provider type, so you have to specify when a provider subclass
shares a source with its parent class.

## Suitability

The first question to ask about a new provider is where it will be
functional, which Puppet describes as suitable. Unsuitable
providers cannot be used to do any work, although we're working on
making the suitability test late-binding, meaning that you could
have a resource in your configuration that made a provider
suitable. If you start puppetd or puppet in debug mode, you'll see
the results of failed provider suitability tests for the resource
types you're using.

Puppet providers include some helpful class-level methods you can
use to both document and declare how to determine whether a given
provider is suitable. The primary method is commands, which
actually does two things for you: It declares that this provider
requires the named binary, and it sets up class and instance
methods with the name provided that call the specified binary. The
binary can be fully qualified, in which case that specific path is
required, or it can be unqualified, in which case Puppet will find
the binary in the shell path and use that. If the binary cannot be
found, then the provider is considered unsuitable. For example,
here is the header for the dpkg provider (as of 0.23.0):

    commands :dpkg => "/usr/bin/dpkg"
    commands :dpkg_deb => "/usr/bin/dpkg-deb"
    commands :dpkgquery => "/usr/bin/dpkg-query"

In addition to looking for binaries, Puppet can compare Facter
facts, test for the existence of a file, or test whether a given
value is true or false. For file extistence, truth, or false, just
call the confine class method with exists, true, or false as the
name of the test and your test as the value:

    confine :exists => "/etc/debian_release"
    confine :true => Puppet.features.rrd?
    confine :false => Puppet.features.rails?

To test Facter values, just use the name of the fact:

    confine :operatingsystem => [:debian, :solaris]
    confine :puppetversion => "0.23.0"

Note that case doesn't matter in the tests, nor does it matter
whether the values are strings or symbols. It also doesn't matter
whether you specify an array or a single value --- Puppet does an OR
on the list of values.

## Default Providers

Providers are generally meant to be hidden from the users, allowing
them to focus on resource specification rather than implementation
details. Toward this end, Puppet does what it can to choose an
appropriate default provider for each resource type.

This is generally done by a single provider declaring that it is
the default for a given set of facts, using the defaultfor class
method. For instance, this is the apt provider's declaration:

    defaultfor :operatingsystem => :debian

The same fact matching functionality is used, so again case does
not matter.

## Provider/Resource API

Providers never do anything on their own; all of their action is
triggered through an associated resource (or, in special cases,
from the transaction). Because of this, resource types are
essentially free to define their own provider interface if
necessary, and providers were initially developed without a clear
resource/provider API (mostly because it wasn't clear whether such
an API was necessary or what it would look like). At this point,
however, there is a default interface between the resource type and
the provider.

This interface consists entirely of getter and setter methods. When
the resource is retrieving its current state, it iterates across
all of its properties and calls the getter method on the provider
for that property. For instance, when a user resource is having its
state retrieved and its uid and shell properties are being managed,
then the resource will call uid and shell on the provider to figure
out what the current state of each of those properties is. This
method call is in the retrieve method in Puppet::Property.

When a resource is being modified, it calls the equivalent setter
method for each property on the provider. Again using our user
example, if the uid was in sync but the shell was not, then the
resource would call shell=(value) with the new shell value.

The transaction is responsible for storing these returned values
and deciding which value to actually send, and it does its work
through a PropertyChange instance. It calls sync on each of the
properties, which in turn just call the setter by default.

You can override that interface as necessary for your resource
type, but in the hopefully-near future this API will become more
solidified.

Note that all providers must define an instances class method that
returns a list of provider instances, one for each existing
instance of that provider. For instance, the dpkg provider should
return a provider instance for every package in the dpkg database.

## Provider Methods

By default, you have to define all of your getter and setter
methods. For simple cases, this is sufficient --- you just implement
the code that does the work for that property.

However, because things are rarely so simple, Puppet attempts to
help in a few ways.

### Prefetching

First, Puppet transactions will prefetch provider information by
calling prefetch on each used provider type. This calls the
instances method in turn, which returns a list of provider
instances with the current resource state already retrieved and
stored in a @property\_hash instance variable. The prefetch method
then tries to find any matching resources, and assigns the
retrieved providers to found resources. This way you can get
information on all of the resources you're managing in just a few
method calls, instead of having to call all of the getter methods
for every property being managed. Note that it also means that
providers are often getting replaced, so you cannot maintain state
in a provider.

### Resource Methods

For providers that directly modify the system when a setter method
is called, there's no substitute for defining them manually. But
for resources that get flushed to disk in one step, such as the
ParsedFile providers, there is a mk\_resource\_methods class method
that creates a getter and setter for each property on the resource.
These methods just retrieve and set the appropriate value in the
@property\_hash variable.

### Flushing

Many providers model files or parts of files, so it makes sense to
save all of the writes up and do them in one run. Providers in need
of this functionality can define a flush instance method to do
this. The transaction will call this method after all values are
synced (which means that the provider should have them all in its
@property\_hash variable) but before refresh is called on the
resource (if appropriate).
