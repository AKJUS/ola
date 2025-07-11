OSC Plugin Developer Information
================================

Welcome to the OSC Plugin walkthrough. This is a step by step guide on how
to write a plugin for OLA, using the OSC (Open Sound Control) plugin as an
example. The OSC plugin uses [liblo](http://liblo.sourceforge.net/) to
handle the OSC messages.

This assumes you have a working knowledge of C++ including references and
pointers as well as a basic understanding of the string, map and vector
classes. Most of the documentation is inline with the code so please keep it
up to date if you change anything.

Before we begin you should do the following:

* Read https://wiki.openlighting.org/index.php/OLA_developer_info which
  provides an overview of some of the classes we'll use.
* Have an understanding of the map & vector classes from the STL
  (http://www.sgi.com/tech/stl/ or http://www.cplusplus.com/reference/stl/).
* Know something about the
  [OSC specification](https://opensoundcontrol.stanford.edu/spec-1_0.html). You
  don't need to know the wire format for the OSC messages. but you should at
  least recognize OSC addresses and know the different data types in a message.
* Look at the [liblo site](http://liblo.sourceforge.net/) and read the
  examples.
* Get familiar with the
  [string class](http://www.cplusplus.com/reference/string/string/), since
  we use that a bit.
* Basic knowledge of shell scripting, which is used in the `./configure`
  script.
* Make sure you can complete a full OLA build & test cycle (`make`,
  `make check`).

The initial commit of this plugin was Rev
`b9d6a47e9a2586d51dac521d7532fb11d76e0ae5`. It's worth having that diff
open as you step through the explanations.

Let's get to it. If you make it all the way through this doc and I ever meet
you in person I'll buy you a beer!


Chapter 1, Build Configuration
==============================

Before we start writing code, we need to configure the build system. Start
off by creating the `plugins/osc` directory.

1.1 Configure script
--------------------

Next we need to modify the `./configure` script so that we only build the
OSC plugin if liblo is installed. So that we don't break existing users of
OLA, a system without liblo will just skip building the plugin, rather than
causing a failure in the configure script.

To be polite, we should also provide a `./configure` flag to disable the osc
plugin, even if liblo is installed.

The `./configure` is generated from `configure.ac` so that's where we'll do
our changes. `configure.ac` is essentially shell script.

    AC_ARG_ENABLE(
      [osc],
      AS_HELP_STRING([--disable-osc],
                     [Disable the OSC plugin even if liblo exists]))

These lines add a `./configure` argument `--disable-osc`. If this
argument is provided, the variable `$enable_liblo` will be set to "no".

    have_liblo="no"

This defaults the `$have_liblo` variable to "no".

    if test "${enable_osc}" != "no"; then
      PKG_CHECK_MODULES(liblo, [liblo >= 0.26], [have_liblo="yes"], [true])
    fi

If the `enable_liblo` variable does not equal "no" (i.e. `--disable-osc`)
wasn't provided, we go ahead and run `pkg-config` to check that liblo is
installed. The `PKG_CHECK_MODULES()` macro does the equivalent of:

    $ pkg-config --modversion liblo

To ensure the version of liblo is >= 0.26, and if it's installed, it sets
the `have_liblo` to "yes" and the automake variables `liblo_CFLAGS` and
`liblo_LIBS` from the output of:

    $ pkg-config --cflags --libs liblo

We'll use `liblo_CFLAGS` and `liblo_LIBS` in the Makefile.

    AM_CONDITIONAL(HAVE_LIBLO, test "${have_liblo}" = "yes")

This sets the automake variable `HAVE_LIBLO` according to the value of
`have_liblo`. This is what controls if we build the osc plugin.

    if test "${have_liblo}" = "yes"; then
      PLUGINS="${PLUGINS} osc"
      AC_DEFINE(HAVE_LIBLO, 1, [define if liblo is installed])
    fi

Finally if `have_liblo` is "yes", we append osc to the `PLUGINS` variable
and set a `#define HAVE_LIBLO 1`. Variables specified in `AC_DEFINE` are
written to the `config.h` file which can be included as a normal header
file. This allows the plugin loader code to know whether to attempt to load
the osc plugin.

Finally don't forget to add `plugins/osc/Makefile` to the `AC_OUTPUT`
section at the end. The tells autoconf to write a Makefile for the
`plugins/osc` directory.

To summarize, we know have the following:

* The automake variable `HAVE_LIBLO`, which is true if we should build the
  OSC plugin.
* The automake variables `liblo_CFLAGS` & `liblo_LIBS`, which contains the
  compiler and linker options required to use liblo.
* A `#define`d value `HAVE_LIBLO` which lets us know within the code if the
  OSC plugin was built.

1.2 Makefiles
-------------

On to the Makefiles. Because we use autoconf / automake the Makefile is
generated from `Makefile.am`. First we edit `plugins/Makefile.am` and add
the `osc` directory to the `SUBDIRS` line.

Now we can create a `plugins/osc/Makefile.am`. We'll cover this file line by
line.

    include $(top_srcdir)/common.mk

This does what you'd expect, it includes the top level `common.mk` file,
which defines `$(COMMON_CXXFLAGS)` to include things like `-Wall` and 
`-Werror`.

    libdir = $(plugindir)

This sets the destination install directory for the shared object (the OSC
plugin in this case) to the plugin directory. By default that's
`/usr/local/lib/olad` but the user may have changed it.

    EXTRA_DIST = OSCAddressTemplate.h OSCDevice.h OSCNode.h OSCPlugin.h \
                 OSCPort.h OSCTarget.h

This specifies the header files that need to be included in the tarball.

    if HAVE_LIBLO

Everything from this point onwards is only run if `HAVE_LIBLO` was true.

    noinst_LTLIBRARIES = libolaoscnode.la
    libolaoscnode_la_SOURCES = OSCAddressTemplate.cpp OSCNode.cpp
    libolaoscnode_la_CXXFLAGS = $(COMMON_CXXFLAGS) $(liblo_CFLAGS)
    libolaoscnode_la_LIBADD = $(liblo_LIBS)

This sets up our first build target, a shared library called
`libolaoscnode.la`. This library contains the `OSCNode` class, along with
the functions in `OSCAddressTemplate.cpp`. The `libolaoscnode.la` is a
helper library, it's not installed when the user runs `make install`(the
`noinst_` prefix does this) but it simplifies the build rules since both the
OSC plugin and the tests depend on it. Because we depend on liblo we need to
add `liblo_CFLAGS` and `liblo_LIBS` to the appropriate lines.

    lib_LTLIBRARIES = libolaosc.la
    libolaosc_la_SOURCES = OSCDevice.cpp OSCPlugin.cpp
    libolaosc_la_CXXFLAGS = $(COMMON_CXXFLAGS) $(liblo_CFLAGS)
    libolaosc_la_LIBADD = libolaoscnode.la

Here is our `llibolaosc.la` plugin. This contains code from `OSCDevice.cpp`
and `OSCPlugin.cpp`. Note that because the code in `OSCPort` is simple, all
the code lives in `OSCPort.h` and there is no corresponding `.cpp` file. If
there was we'd need to list it here as well. This depends on the
`libolaoscnode.la` helper library.

    if BUILD_TESTS
      TESTS = OSCTester
    endif

On to the tests, These three lines add the OSCTester program to the list of
tests to execute when `make check` is run. If the user ran `./configure`
with `--disable-unittests` we don't build or run any of the unittesting
code.

    check_PROGRAMS = $(TESTS)

This sets `check_PROGRAMS` to what's contained in the `$TESTS` variable.

    OSCTester_SOURCES = OSCAddressTemplateTest.cpp \
                        OSCNodeTest.cpp \
                        OSCTester.cpp
    OSCTester_CXXFLAGS = $(COMMON_CXXFLAGS) $(CPPUNIT_CFLAGS)
    OSCTester_LDADD = $(CPPUNIT_LIBS) \
                      libolaoscnode.la \
                      ../../common/libolacommon.la \
                      ../../common/testing/libolatesting.la

Here are the rules for OSCTester. This specifies the 3 source files as well
as the libraries the code depends on.

    endif

This completes the `if HAVE_LIBLO` from above.

1.3 Putting it together
-----------------------

At this point you can move back to the top level directory and run the
following:

    autoreconf

This creates the `./configure` script, which we'll now run:

    ./configure

If you watch the output you should see the line "checking for liblo... yes".
You should also see that `HAVE_LIBLO` is `#define`d to 1 in the `config.h`
file and that the `plugins/osc/Makefile` has been created.

That's it for the build configuration. Time to write some code!

Chapter 2, Plugin Boilerplate
=============================

2.1 Plugin ID
-------------

We need to reserve a plugin ID in `common/protocol/Ola.proto`. Please see the
note in that file regarding how we coordinate assigning plugin IDs.

Once we've assigned a plugin id, we can run `make` in the top directory. This
will update `include/ola/plugin_id.h`. You'll notice `plugin_id.h` isn't
included in the git repo since it's a generated file.

2.2 Plugin Loader
-----------------

We need to tell the Plugin Loader to load our new OSC plugin. Remember the
`HAVE_LIBLO` variable in `config.h`? We'll make use of that now.

In `olad/DynamicPluginLoader.cpp` we add:

    #ifdef HAVE_LIBLO
    #include "plugins/osc/OSCPlugin.h"
    #endif  // HAVE_LIBLO

`DynamicPluginLoader.cpp` includes `config.h` at the top of the file, which
is how `HAVE_LIBLO` is defined.

    #ifdef HAVE_LIBLO
      m_plugins.push_back(
          new ola::plugin::osc::OSCPlugin(m_plugin_adaptor));
    #endif  // HAVE_LIBLO

If `HAVE_LIBLO` is defined, we go ahead and add the OSCPlugin to the vector
of plugins to load.

That's it for the boilerplate.


Chapter 3, Helper Code
======================

The OSC code is divided into two parts:

* OSCNode is a C++ & DMX orientated wrapper around liblo.
* OSCPlugin, OSCDevice & OSCPort are the glue between OLA and the OSCNode.

There is a little bit of support code that is used in both parts, so we
factor that out here.

3.1 OSCTarget
-------------

`OSCTarget.h` contains a struct used to represent OSC targets. Each target
has an IP & UDP Port, along with a string that holds the OSC address for the
target. We also provide three constructors to help us create `OSCTarget`
structs. The first is the default constructor, the second is the copy
constructor and the third takes an `IPV4SocketAddress` and string and
initializes the member variables.

Notice in the second constructor we pass both the `IPV4SocketAddress` and
string by (const) reference. If we passed by value (the default) we'd incur
a copy of the `IPV4SocketAddress` / string class. A good string
implementation would avoid copying the actual string data and rather just
copy the internal members of the string class. However even that is likely
to be more expensive than taking a reference, so we pass strings by const
reference wherever possible.

You'll notice that all the OSC code is contained within the
`ola::plugin::osc` namespace. This is to avoid conflicts with the core OLA
classes and code in other plugins.

3.2 ExpandTemplate()
--------------------

`OSCAddressTemplate.h` provides a single method, `ExpandTemplate()` which
behaves a bit like `printf`. Why don't we just use `printf` you ask? Because
the strings to expand are provided in the config file by the user and using
`printf` for user specified strings is dangerous (see
http://www.cis.syr.edu/~wedu/Teaching/cis643/LectureNotes_New/Format_String.pdf).

At this point it's worth mentioning testing. Have a look at the
`OSCAddressTemplateTest.cpp` file. We define a single test method
`testExpand` that confirms `ExpandTemplate()` behaves as we expect.

If you're in the mood for experimentation, change

    output.find("%d");

to

    output.find_first_of("%d");

Watch the tests fail and make sure you understand why.


Chapter 4, OSCNode
==================

As mentioned, OSCNode is the C++ wrapper around liblo. It uses the OLA
network & IO classes so that it integrates nicely with OLA. As one would
expect, the code is split across `OSCNode.h` and `OSCNode.cpp`.

4.1 `OSCNode.h`
---------------

Lets step through the interface to the `OSCNode` class. First we include all
the headers we're going to depend on. Since all our code exists in the
`ola::plugin::osc` namespace, we need to either add `using ...` directives
or fully qualify the data types in the rest of the file. Generally if I'm
going to use a data type more than once in a file I'll typically add a
`using` directive.

Options Structure
-----------------

The `OSCNode` constructor takes a const reference to a `OSCNodeOptions`
structure. Often the number of options passed to a constructor grows over
the lifetime of the code. Consider a class `Foo`, which starts off with:

    Foo(const string &option1, int option2);

Then at some later date, someone added another int and a bool option:

    Foo(const string &option1, int option2, int option3, bool option4);

At this point you need to update all the caller of `Foo(...)`. It gets even
worse we people start adding default values:

    Foo(const string &option1, int option2, int option3, bool option4,
        bool option5 = false, bool option6 = true);

Now if the caller wants to set `option6` to false, they need to set
`option5` as well. This leads to buggy code.

Passing in an options struct avoids this. The constructor can set the
default values for all options and then the callers can override the options
they are interested in. More options can then be added at a later stage
without having to update all the callers.

The downside to using an option struct is that mandatory options can be
omitted, We could add a `IsValid()` method on the options struct to check
for this, but it's not worth it in this case.

Ok, back to the code, the next line typedefs a
`Callback1<void, const DmxBuffer&>` to `DMXCallback`. This shortens the code
a bit, shorter code is usually easier to understand.

The `OSCNode` constructor takes a `SelectServer`, `ExportMap` as well as the
previously described options struct.

Methods
-------

The `Init()` and `Stop()` methods are self explanatory.

Sending Data
------------

The next three methods (`AddTarget`, `RemoveTarget` & `SendData`) are used
for sending DMX data using OSC. To ease the burden on the caller, we group
`OSCTarget`s into groups (one target may exist in more than one group but
that's rare). In OLA, each group can be thought of as a universe.

Groups are identified by their `group_id`, which is just an integer.
`AddTarget(id, target)` associates a target with a group. `RemoveTarget`
does the opposite. `SendData(id, dmx_data)` sends the DMX data to all
members of the group.

Receiving Data
--------------

The `RegisterAddress()` method is used for receiving DMX data over OSC. It
takes an `osc_address` (e.g. `/dmx/universe/1`), and a callback. Whenever
OSC data is received for the given address, the callback is invoked and
passed the DMX data.

When the callback argument to `RegisterAddress()` is `NULL` the
`osc_address` is un-registered.

Finally `HandleDMXData()` is a method called by liblo.

Private Types, Variables & Methods
----------------------------------

Let's go over the internal data members of the `OSCNode` class.

`NodeOSCTarget` is derived from `OSCTarget`. It includes a `lo_address`
member. We do this because liblo doesn't provide a way to create
`lo_address`es from a socket address, rather `lo_address_new()` takes
strings, and then calls `inet_aton` to convert to the network data types. We
don't want to incur this overhead ever time we send a packet, so we create
the `lo_address` object once for each target in `AddTarget()`.

Next we have three typedefs, which are used with our internal data
structures. On the sending side, we need to track a collection of
`OSCTarget`s for each group. We do this with a map of vectors, so you can
think of it like this:

    Group 1 ->
      [ target1, target2, target3 ],
    Group 2 ->
      [ target4 ]

We could have used a set rather than a vector to hold the `NodeOSCTarget`
objects, but then we'd need to define a `<` operator since the set container
stores it's elements in sorted order. Using a set would also require us to
store the objects directly rather than pointers to the objects, This would
incur additional copying when elements are inserted into the set but since
we only do this once when the plugin starts and the number of elements (OSC
Targets) is small it wouldn't matter.

We typedef a vector of pointers to `NodeOSCTarget`s to `OSCTargetVector`,
and then typedef a map of unsigned ints to pointers to `OSCTargetVector`s as
a `OSCTargetMap`.

On the receiving side, we have a map of OSC addresses to callbacks:

    "/dmx/universe/8" ->
      <Callback Object>
    "/dmx/universe/9 ->
      <Callback Object>

We typedef a map of strings to callback objects to `AddressCallbackMap`.

That's it for the types, the members are documented in the `OSCNode.h` file
itself. We store pointers to the `SelectServer` and `ExportMap` objects
passed in to the constructor as well as the UDP port to listen on.

We also hold a pointer to a `UnmanagedFileDescriptor` which is how we add
the socket description used by liblo to the `SelectServer`. There is also
the `lo_server` data, as well as the `OSCTargetMap` and `AddressCallbackMap`
discussed above. On Windows, we need to create an
`UnmanagedSocketDescriptor` instead.

The `DescriptorReady` method is called when the liblo socket description has
data pending.

Finally there are two static variables, `DEFAULT_OSC_PORT` which is the
default UDP port to listen on and `OSC_PORT_VARIABLE`, which is what we use
with the `ExportMap`.

4.2 `OSCNode.cpp`
-----------------

Onwards to the implementation! Most of this is documented in the file itself
so I'll just point out the interesting bits.

    void OSCErrorHandler(int error_code, const char *msg, const char *stack)

The call to `lo_server_new_with_proto()` takes a pointer to a function which
is called when liblo wants to log an error message. We pass a pointer to
`OSCErrorHandler` which uses the OLA logging mechanism.

`OSCDataHandler` is what liblo calls when there is new OSC data. The last
argument is a pointer back to the `OSCNode` object, so after we perform some
checks we call `HandleDMXData()`. We return 0 so that liblo doesn't try to
find other OSC handlers to process the same data (see the liblo API for more
info on this).

In the `OSCNode` constructor, if an `ExportMap` was provided, we export the
UDP port that liblo is listening on. If olad is built with the web server
then users can navigate to http://<ip>:9090/debug and see the internal state
of the server. The `ExportMap` provides an easy method to export this
internal state, and we should really make more use of it since it's
incredibly valuable for debugging.

The destructor just calls `Stop()`. Note that `Stop()` may have been called
previously, so we need to account for that.

The `Init` method sets up the node. It creates a new `lo_server` and
associates the socket description with the `SelectServer`.

`Stop` deletes any of the entries in the `OSCTargetMap` and
`AddressCallbackMap`, cleans up the socket description and frees the
`lo_server`. It's important to clear both maps once the entries have been
deleted since `Stop()` may be called more than once. Similarly, once the
`UnmanagedFileDescriptor` and `lo_server` have been freed we set both of
them to `NULL`.

`SendData()` packs the `DmxBuffer` into a `lo_blob` variable. We use OSC
blobs for transporting DMX over OSC. It then calls `lo_send_from` for each
target in the group.


Chapter 5, Testing
==================

Congratulations, at this point we're more than half way through. Let's take
a detour and discuss about testing.

All good code comes with tests and OLA is no exception. Well written tests
increases the chance of having your plugin accepted into OLA. The
`OSCNodeTest.cpp` contains the unittests for the `OSCNode` class.

To test the node, we create a second UDP socket (liblo creates the first
one) which we then use to send and receive OSC messages. For each test we
register a timeout which aborts the test after `ABORT_TIMEOUT_IN_MS`. This
prevents the test from hanging if the packets are dropped (which they
shouldn't be since it's all localhost communication).

In this test code we just use `OLA_ASSERT_TRUE`. `ola/testing/TestUtils.h`
provides many more testing macros so take a look.

Finally we need the `OSCTester.cpp` to pull it all together. This is just
boiler plate code that sets up the unittests.


Chapter 6, Plugins, Devices & Ports
===================================

Now it's just a matter of building the OLA classes around the `OSCNode`. In
OLA a Plugin creates objects of the Device type (if you use Java or other
object orientated languages think of the Plugin class as a factory). Each
device then has one of more input and output ports. For the OSC plugin, we
allow the user to specify the number of input and output ports to create in
the config file.

Each input port is assigned an OSC address (e.g. `/dmx/universe` or
`/foo/bar`). If `%d` occurs in the OSC address it's replaced
by the current universe number. The default OSC address for each port is
`/dmx/universe/%d` so the default behavior is sane.

Each output port is configured with a list of OSC Targets in the form
`ip:port/address`.

6.1 Plugin
----------

The OSCPlugin is pretty simple. Like all OLA plugins, it inherits from the
base `ola::Plugin` class. It's passed a pointer to a `PluginAdaptor` object,
which is how the plugins interface with the rest of OLA.

We'll cover the methods in a different order from which they appear in the
file, since I think it makes more sense.

`Description()` just returns a string which contains a description for the
Plugin. This is what appears in the output of `ola_plugin_info` or on the
web page under the 'Plugins' menu. It should explain what the plugin does,
as well as the configuration options available.

`SetDefaultPreferences()` is used to initialize the config file. It should
check if configuration options exist, and if they don't reset them to
default values. We use a bool to keep track if any of the options have
changed, since if they do we need to write out the changes.

Once `SetDefaultPreferences()` is called, the next method to run is
`StartHook()`. This reads the configuration options, extracts the
`OSCTarget`s from the string (by calling `ExtractOSCTarget()`) and adds them
to a vector.

Once the configuration options have been processed we create a new
`OSCDevice`, call `Start()` and register it with OLA.

The `StopHook()` method unregisters the device and deletes it.

`GetPortCount()` is a helper method to save duplicating similar code more
than once.

`ExtractOSCTarget` unpacks a string in the form `ip:port/address` into a
`OSCTarget` structure.

6.2 Device
----------

Like the Plugin, the device is also pretty simple. Each `OSCDevice` owns a
`OSCNode` object. In the constructor, the `OSCDevice` takes a pointer to the
plugin which created the device, as well as the UDP port to listen on, and a
list of addresses (for the input ports) and targets (for the output ports).
These data structures are saved as member variables since they are required
in `StartHook()`.

The `OSCDevice` uses the static ID of 1, which means we can only ever have
one `OSCDevice`. This is probably fine unless people are doing fancy things
with multiple network interfaces.

`AllowMultiPortPatching()` returns true, which means that multiple ports of
the same type can be patched to the same universe. This is fine because the
`RegisterAddress` method in `OSCNode` detects if we try to register the same
address more than once.

The only interesting method is `StartHook()`. This calls `Init` on the
`OSCNode` and then sets up the ports. We call `Init()` from within
`StartHook()` rather than in the constructor so that we can propagate a
failure up the call change (constructors don't have return values so there
is no way to notify the caller of an error). In general it's best to avoid
any code that may fail in a constructor.

`StartHook` iterates over the vector of addresses and creates an
`OSCInputPort` for each. Similarly it iterates over the vector of vectors
contains `OSCTarget`s and sets up an `OSCOutputPort` for each.

For both the input and output ports we create a description string which
shows up in `ola_dev_info` and on the web UI. For the input port this is the
OSC address they are listening on, for the output ports it's the list of OSC
targets it's sending to.

6.3 Ports
---------

Since there isn't much code for the `Port` classes I put it all in
`OSCPort.h` rather than splitting over two files.

The `OSCInputPort` is the more complex of the two. Firstly the
`PreSetUniverse()` method is called, which is where we register a callback
with the `OSCNode` (if needed). If the registration fails (say another port
has the same address) we return false, and OLA informs the user that
patching failed.

We also need to handle the unpatch case, where the `new_universe` argument
will be `NULL`. In this case we call `RegisterAddress` and pass a `NULL` as
the callback argument which unregisters the address.

The callback passed to `RegisterAddress()` runs the `NewDMXData()` method,
which copies the `DmxBuffer` values into the port's internal buffer and
calls `DmxChanged()`. This notifies the universe that the port is bound to
that new data has arrived. The universe then calls `ReadDMX()` to collect
the new data.

`OSCOutputPort` is much more simple, when the `WriteDMX()` method is called
we call `SendData()` on the `OSCNode` object. Note that our simple DMX over
OSC format has no notion of priority, so we ignore the priority argument.


Chapter 7, Wrapping Up
======================

That's it for the code. At this point you can change back to the top level
directory and run `make` to build the new plugin. Running `make check` will
run all the unittests.

While you're developing, you'll find it quicker to run `make` and
`make check` within the directory you're changing. Just remember to run it
at the top level every so often or before trying to run olad.

Chapter 8, Final Words
======================

8.1 Code Style
--------------

Just a couple more points to wrap things up. As mentioned into
README.developer, OLA uses the [Google C++ style guide](http://google-styleguide.googlecode.com/svn/trunk/cppguide.xml)
There is a tool, cpplint.py which will check the code for style violations.
See README.developer for more information.

8.2 Memory Leaks
----------------

Valgrind is an excellent tool for discovering and debugging memory leaks.
You can find out more at http://valgrind.org/.
