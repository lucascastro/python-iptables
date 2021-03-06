Examples
========

Rules
-----

In ``python-iptables``, you usually first create a rule, and set any
source/destination address, in/out interface and protocol specifiers, for
example::

    >>> import iptc
    >>> rule = iptc.Rule()
    >>> rule.in_interface = "eth0"
    >>> rule.src = "192.168.1.0/255.255.255.0"
    >>> rule.protocol = "tcp"

This creates a rule that will match TCP packets coming in on eth0, with a
source IP address of 192.168.1.0/255.255.255.0.

A rule may contain matches and a target. A match is like a filter matching
certain packet attributes, while a target tells what to do with the packet
(drop it, accept it, transform it somehow, etc). One can create a match or
target via a Rule::

    >>> rule = iptc.Rule()
    >>> m = rule.create_match("tcp")
    >>> t = rule.create_target("DROP")

Match and target parameters can be changed after creating them. It is also
perfectly valid to create a match or target via instantiating them with
their constructor, but you still need a rule and you have to add the matches
and the target to their rule manually::

    >>> rule = iptc.Rule()
    >>> match = iptc.Match(rule, "tcp")
    >>> target = iptc.Target(rule, "DROP")
    >>> rule.add_match(match)
    >>> rule.target = target

Any parameters a match or target might take can be set via the attributes of
the object. To set the destination port for a TCP match::

    >>> rule = iptc.Rule()
    >>> rule.protocol = "tcp"
    >>> match = rule.create_match("tcp")
    >>> match.dport = "80"

To set up a rule that matches packets marked with 0xff::

    >>> rule = iptc.Rule()
    >>> rule.protocol = "tcp"
    >>> match = rule.create_match("mark")
    >>> match.mark = "0xff"

Parameters are always strings.

When you are ready constructing your rule, add them to the chain you want it
to show up in::

    >>> chain = iptc.Chain(iptc.Table(iptc.Table.FILTER), "INPUT")
    >>> chain.insert_rule(rule)

This will put your rule into the INPUT chain in the filter table.

Chains and tables
-----------------

You can of course also check what a rule's source/destination address,
in/out inteface etc is. To print out all rules in the FILTER table::

    >>> import iptc
    >>> table = iptc.Table(iptc.Table.FILTER)
    >>> for chain in table.chains:
    >>>     print "======================="
    >>>     print "Chain ", chain.name
    >>>     for rule in chain.rules:
    >>>         print "Rule", "proto:", rule.protocol, "src:", rule.src, "dst:", \
    >>>               rule.dst, "in:", rule.in_interface, "out:", rule.out_interface,
    >>>         print "Matches:",
    >>>         for match in rule.matches:
    >>>             print match.name,
    >>>         print "Target:",
    >>>         print rule.target.name
    >>> print "======================="

As you see in the code snippet above, rules are organized into chains, and
chains are in tables. You have a fixed set of tables; for IPv4::

* FILTER,
* NAT,
* MANGLE and
* RAW.

For IPv6 the tables are::

* FILTER,
* MANGLE,
* RAW and
* SECURITY.

To access a table::

    >>> import iptc
    >>> table = iptc.Table(iptc.Table.FILTER)
    >>> print table.name
    filter

To create a new chain in the FILTER table::

    >>> import iptc
    >>> table = iptc.Table(iptc.Table.FILTER)
    >>> chain = table.create_chain("testchain")

    $ sudo iptables -L -n
    [...]
    Chain testchain (0 references)
    target     prot opt source               destination

To access an existing chain::

    >>> import iptc
    >>> table = iptc.Table(iptc.Table.FILTER)
    >>> chain = iptc.Chain(table, "INPUT")
    >>> chain.name
    'INPUT'
    >>> len(chain.rules)
    10
    >>>

More about matches and targets
------------------------------

There are basic targets, such as ``DROP`` and ``ACCEPT``. E.g. to reject
packets with source address ``127.0.0.1/255.0.0.0`` coming in on any of the
``eth`` interfaces::

    >>> import iptc
    >>> chain = iptc.Chain(iptc.Table(iptc.Table.FILTER), "INPUT")
    >>> rule = iptc.Rule()
    >>> rule.in_interface = "eth+"
    >>> rule.src = "127.0.0.1/255.0.0.0"
    >>> target = iptc.Target(rule, "DROP")
    >>> rule.target = target
    >>> chain.insert_rule(rule)

To instantiate a target or match, we can either create an object like above,
or use the ``rule.create_target(target_name)`` and
``rule.create_match(match_name)`` methods. For example, in the code above
target could have been created as::

    >>> target = rule.create_target("DROP")

instead of::

    >>> target = iptc.Target(rule, "DROP")
    >>> rule.target = target

The former also adds the match or target to the rule, saving a call.

Another example, using a target which takes parameters. Let's mark packets
going to ``192.168.1.2`` UDP port ``1234`` with ``0xffff``::

    >>> import iptc
    >>> chain = iptc.Chain(iptc.Table(iptc.Table.MANGLE), "PREROUTING")
    >>> rule = iptc.Rule()
    >>> rule.dst = "192.168.1.2"
    >>> rule.protocol = "udp"
    >>> match = iptc.Match(rule, "udp")
    >>> match.dport = "1234"
    >>> rule.add_match(match)
    >>> target = iptc.Target(rule, "MARK")
    >>> target.set_mark = "0xffff"
    >>> rule.target = target
    >>> chain.insert_rule(rule)

Matches are optional (specifying a target is mandatory). E.g. to insert a rule
to NAT TCP packets going out via ``eth0``::

    >>> import iptc
    >>> chain = iptc.Chain(iptc.Table(iptc.Table.NAT), "POSTROUTING")
    >>> rule = iptc.Rule()
    >>> rule.protocol = "tcp"
    >>> rule.out_interface = "eth0"
    >>> target = iptc.Target(rule, "MASQUERADE")
    >>> target.to_ports = "1234"
    >>> rule.target = target
    >>> chain.insert_rule(rule)

Here only the properties of the rule decide whether the rule will be applied
to a packet.

Matches are optional, but we can add multiple matches to a rule. In the
following example we will do that, using the ``iprange`` and the ``tcp``
matches::

    >>> import iptc
    >>> rule = iptc.Rule()
    >>> rule.protocol = "tcp"
    >>> match = iptc.Match(rule, "tcp")
    >>> match.dport = "22"
    >>> rule.add_match(match)
    >>> match = iptc.Match(rule, "iprange")
    >>> match.src_range = "192.168.1.100-192.168.1.200"
    >>> match.dst_range = "172.22.33.106"
    >>> rule.add_match(match)
    >>> rule.target = iptc.Target(rule, "DROP")
    >>> chain = iptc.Chain(iptc.Table.(iptc.Table.FILTER), "INPUT")
    >>> chain.insert_rule(rule)

This is the ``python-iptables`` equivalent of the following iptables command::

    # iptables -A INPUT -p tcp –destination-port 22 -m iprange –src-range 192.168.1.100-192.168.1.200 –dst-range 172.22.33.106 -j DROP
