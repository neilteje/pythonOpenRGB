Advanced Usage
==============

Modifying Modes
---------------
If you want to change things like a mode's speed, you will need to edit the mode.
Before you do this, make sure that the property you want to edit is allowed in
that mode.  There are few ways to test this.

*  You can look in the OpenRGB GUI, select the mode you want to edit, and
   check if the option to edit the property is changeable or grayed out.
* You can check if the property you want to change is :code:`None`

.. code-block:: python

    if mode.speed is None:
        print("Speed not supported")
    if mode.direction is None:
        print("Direction not supported")
    ...

* You can check the mode's :any:`ModeFlags`

.. code-block:: python

    if ModeFlags.HAS_SPEED in mode.flags:
        print("Speed supported")
    if ModeFlags.HAS_MODE_SPECIFIC_COLOR in mode.flags:
        print("Mode specific color supported")

.. note::

    All of this is checked when you set the mode, so if you try to set an unsupported value, you'll get an appropriate error message.

After you verify that the property that you want to change is supported by the
mode, you can edit it.  This can be done by directly editing a mode's properties

.. code-block:: python

    rainbow = cli.devices[0].modes[3]
    rainbow.speed = 0
    cli.devices[0].set_mode(rainbow)

.. warning::

	Most properties have a min and a max.  For speed, make sure to set the :code:`mode.speed` value to something between :code:`mode.speed_min` and :code:`mode.speed_max`.

Or more compactly:

.. code-block:: python

    cli.devices[0].modes[2].colors = [RGBColor(255, 0, 0), RGBColor(0, 0, 255)]
    cli.devices[0].set_mode(2)


Optimizing for Speed
--------------------
For creating custom effects, you often want to have your lights refreshing
quickly.  Optimizing openrgb-python for speed isn't very hard if you know what
you are doing.  The best way to maximize speed is to minimize OpenRGB SDK calls.
The most user-friendly way to do this is to use the
:doc:`effects control flow</pages/effects>`, but it is possible to do
accomplish the same things with the other functions.  One common argument that
will help is the :code:`fast` argument.  In any of the color-changing functions
(:code:`set_color`, :code:`set_colors`, :code:`show`), if you pass in
:code:`True` to the :code:`fast` argument it will sacrifice internal state
management for speed.  The way it does this is by skipping the
:any:`RGBObject.update` call.  This is fine to use as long as you know what it
does, or rather doesn't do.

Minimizing SDK calls means converting code like this...

.. code-block:: python

    cli.devices[0].set_color(RGBColor(255, 0, 0))
    cli.devices[0].set_color(RGBColor(0, 255, 0), 4, 8)
    cli.devices[0].zones[0].set_color(RGBColor(0, 0, 255))

... into code like this (assuming the first zone is 3 LEDs long)

.. code-block:: python

    cli.devices[0].set_colors(
        [RGBColor(0, 0, 255)]*3 \
        + [RGBColor(255, 0, 0)] \
        + [RGBColor(0, 255, 0)]*4
    )

Which is pretty ugly, but only uses one SDK call, which makes it faster.  The
alternative control method is basically a better looking way of writing the
above code.

.. code-block:: python

    cli.devices[0].colors = [RGBColor(255, 0, 0)]*8
    cli.devices[0].colors[:3] = [RGBColor(0, 0, 255)]*3
    cli.devices[0].colors[4:8] = [RGBColor(0, 255, 0)]*4
    cli.devices[0].show()

Controlling the SDK Connection
------------------------------
For background programs, or programs that deal with an OpenRGB SDK server that
is not always on, you might want to start and stop the connection to the SDK
sometimes.

.. code-block:: python

    cli.disconnect()
    time.sleep(5)
    cli.connect()

When dealing with an unreliable SDK server (you don't know when it is on), you
will probably want some error handling.  If there is no SDK server running and
you try to call :any:`OpenRGBClient.connect()` or try to initialize an
:any:`OpenRGBClient`, then a :code:`ConnectionRefusedError` will be raised.  If
the client loses connection to the SDK server after the initial connection, then
trying to interact with the SDK server will cause an :any:`OpenRGBDisconnected`
error.

Offline Profile Editing
-----------------------
Binary OpenRGB profiles, with the '.orp' suffix, can be loaded directly into
OpenRGB-Python as a :any:`Profile` object if you want to inspect or edit an
existing profile.

.. note::

    This doesn't require a connection the OpenRGB SDK server

.. code-block:: python

    from openrgb.utils import Profile

    # Loading a Profile object
    with open('/path/to/profile.orp', rb) as f:
        my_profile = Profile.unpack(f)

    # Modifying the profile
    my_profile.controllers[0].colors[0] = RGBColor(255, 0, 0)

    # Saving a profile
    with open('/path/to/new_profile.orp', 'wb') as f:
        f.write(my_profile.pack())

SDK Protocol Version
--------------------
OpenRGB implemented a versioning system for the SDK protocol, to allow
new features to be added to the SDK protocol without breaking existing SDK
apps.  The versioning system is bidirectionally backwards compatible, so older
SDK apps will work with the newest version of OpenRGB and newer SDK apps will
work with older versions of OpenRGB.  OpenRGB-Python should stay pretty
up-to-date with the latest SDK protocol versions, but if, for whatever reason,
you need to use a specific protocol version, this is how.

.. note::

    When using a newer SDK client that supports versioning with an older version
    of the OpenRGB server, there will be a 1 second delay on initialization
    because a timeout has to be exceeded.

.. code-block:: python

    # To change the protocol version at initialization
    cli = OpenRGBClient(protocol_version=0)

    # To change the protocol version during usage
    cli.protocol_version = 1

In both cases, a :any:`ValueError` will be thrown if the protocol version is
above the highest version supported by both the server and the client.
