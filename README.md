Ansible role for RouterOS
=========================

This role enables easy configuration of Mikrotik RouterOS.

The Problem
-----------

Assume you want to have the following config on your Mikrotik device:

    /ip address add interface=ether1 address=192.168.1.1/24

That's easy to put into [community.routeros.command]. But you won't know if the
command changed anything.

Assume further that you want to change the address from its previous value
`192.168.0.1/24`. You could insert another line, removing the old address:

    /ip address rem [find where interface=ether1 address=192.168.0.1/24]

But what if you don't know the previous value or don't care and simply want to
change the address on interface ether1? You might be tempted to use this then:

    /ip address rem [find where interface=ether1]

But oh no, now you're removing the correct address 192.168.1.1/24 as well!

Also, how do you know whether it was successful?

[community.routeros.command]: https://ansible.fontein.de/collections/community/routeros/command_module.html

The Solution
------------

To achieve the above, you can simply include this role like so:

```yaml
- name: Configure IP address on ether1
  include_role:
    name: routeros
  vars:
    ros_path: ip address
    ros_match_attrs:
      interface: ether1
      dynamic:
        set: no
        value: "no"
    ros_set_attrs:
      address: 192.168.1.1/24
```

The role translates this invocation to a [RouterOS script] that does the
following:

* In `/ip address`, look for an item with `interface=ether1` and `dynamic=no`.
  * If you find it, check if it has `address=192.168.1.1/24`
    * If it has, be done and report no changes.
    * If it doesn't, set its `address=192.168.1.1/24` and report changed state
  * If you don't find it, add one with `interface=ether1 address=192.168.1.1/24`
    and report changed state (note that `dynamic=no` is not set!)
* If there is any unexpected output from RouterOS, report failed state (unless
  `routeros_ignore_errors` is set).

By doing this on the router itself, we save ourselves from several roundtrips
between Ansible and the device.

[RouterOS script]: https://help.mikrotik.com/docs/display/ROS/Scripting

Object placement
----------------

For ordered objects, you can specify `ros_place_before` to define where the
object will be added if it doesn't exist yet:

```yaml
- name: Allow incoming DHCP traffic
  include_role:
    name: routeros
  vars:
    ros_path: ip firewall filter
    ros_match_attr:
      chain: input
      protocol: udp
      dst-port: 67
    ros_set_attr:
      action: accept
    ros_place_before: '[find chain=input action=reject]'
```

You can use RouterOS scripting to find the object to place before dynamically,
or specify an internal id like `*a` that you obtained some other way. Note
however that the scripting expression must return, otherwise the task will fail!
To insert the rule as first object, you can try this:

```
    ros_place_before: '([find]->0)'
```

Note that this doesn't have any influence if the object already exists!

Troubleshooting
---------------

With `-vvv` you can see the generated script that is being sent to the device,
as well as its output. In the output, you will be able to distinguish between
the changed states "added" and "changed".

You should definitely be using [`+512cet`][terminal width hack] at the end of
you username. Still, you might experience situations where the generated script
simply is too long and will cause heaps of "fake" output lines in `-vvv`. That
shouldn't be a problem per se, but makes debugging harder.

[terminal width hack]: https://github.com/ansible-collections/community.routeros/issues/6#issuecomment-720357994

Also, there are situations where the change detection is very fragile. For
example, if setting a time somewhere, you can easily specify it as `10m` to mean
10 minutes. But this will cause a change to be reported every time, because
internally RouterOS converts this to `600`, and while it displays it as `10m`,
it won't match during the search. Nothing is actually changed, but the script
cannot detect that they are the same value.

The same thing applies to arrays: They can be set using their comma-separated
representation, but won't turn up on filtering on that. So the following will
always report changes:

```yaml
- name: Add important script
  include_role:
    name: routeros
  vars:
    ros_path: system script
    ros_match_attr:
      name: test
    ros_set_attr:
      source: ':put "Hi!"'
      policy: "read,write"
```

Instead, you need to write the policy as YaML list:

```yaml
      policy:
      - read
      - write
```

Note that RouterOS is horribly inconsistent with what is an array and what
isn't. For example, `/routing filter set protocol=connect,static` is actually
stored as a string (which means you need to use `"connect,static"` when
filtering for it), whereas `routing filter set address-family=ip,ipv6` is stored
as an array (which means you need to use `("ip","ipv6")` when filtering for it).
