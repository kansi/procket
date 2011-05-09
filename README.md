
procket is an Erlang socket interface which can be used for requesting
socket features that usually require superuser privileges.

procket uses the experimental NIF interface first introduced in Erlang
R13B03.


## CHANGES

### V0.02:
* procket:listen/1,2 has been renamed procket:open/1,2. procket:listen/2
  is now a wrapper around listen(2)

* procket:recvfrom/2 returns {error,eagain} when data is not available
  (previously return nodata)


## EXPORTS

    open(Port, Options) -> {ok, FD} | {error, Reason} | {error, {Reason, Description}}
    
        Types   Port = 0..65535
                Options = [Opts]
                Opts = {pipe, Path} | {protocol, Protocol} | {ip, IPAddress} |
                        {progname, string()} | {interface, string()}
                Protocol = tcp | udp
                IPAddress = string() | tuple()
                Reason = posix()
                Description = string()


## COMPILING

Try running: make


## SETUID vs SUDO vs Capabilities

The procket executable needs root privileges. Either allow your user to
run procket using sudo or copy procket to somewhere owned by root and
make it setuid.

* for sudo

        sudo visudo
        youruser ALL=NOPASSWD: /path/to/procket/priv/procket

* to make it setuid

        sudo cp priv/procket /usr/local/bin
        sudo chown root:yourgroup /usr/local/bin/procket
        sudo chmod 750 /usr/local/bin/procket
        sudo chmod u+s /usr/local/bin/procket

* use Linux capabilities: beam or the user running beam can be
given whatever socket privileges are needed. For example, using file
capabilities:

        setcap cap_net_raw=ep /usr/local/lib/erlang/erts-5.8.3/bin/beam.smp

    To see the capabilities:

        getcap /usr/local/lib/erlang/erts-5.8.3/bin/beam.smp
    
    To remove the capabilities:

        setcap -r /usr/local/lib/erlang/erts-5.8.3/bin/beam.smp


## USING IT

    $ erl -pa ebin
    Erlang R13B03 (erts-5.7.4) [source] [rq:1] [async-threads:0] [hipe] [kernel-poll:false]
    
    Eshell V5.7.4  (abort with ^G)
    1> {ok, FD} = procket:open(53, [{progname, "sudo priv/procket"},{protocol, udp},{type, dgram}]).
    {ok,9}
    2> {ok, S} = gen_udp:open(53, [{fd,FD}]).
    {ok,#Port<0.929>}
    3> receive M -> M end.
    {udp,#Port<0.929>,{127,0,0,1},47483,"hello\n"}
    4>
    
    $ nc -u localhost 53
    hello
    ^C


## EXAMPLES

### Simple echo server

    $ erl -pa ebin
    1> echo:start(53, [{progname, "sudo priv/procket"}, {protocol, tcp}]).

### ICMP ping

    1> icmp:ping("www.yahoo.com").

### Sniff the network

    1> {ok, S} = procket:open(0, [{protocol, 16#0008}, {type, raw}, {family, packet}]).
    {ok,12}
    2> procket:recvfrom(S, 2048).
    {ok,<<0,21,175,89,8,38,0,3,82,3,39,36,8,0,69,0,0,52,242,
              0,0,0,52,6,188,81,209,...>>}
    3> {ok, S1} = gen_udp:open(0, [binary, {fd, S}, {active, false}]).
    4> gen_udp:recv(S1, 2048).

### Bind to one or more interfaces

    1> procket:open(53, [{progname, "sudo priv/procket"},{protocol, udp},{type,dgram},{interface, "br0"}]).
    {ok,9}
    2> procket:open(53, [{progname, "sudo priv/procket"},{protocol, udp},{type,dgram},{interface, "br1"}]).
    {ok,10}


## HOW IT WORKS

procket creates a local domain socket and spawns a small setuid binary
(or runs it under sudo). The executable opens a socket, drops privs and
passes the file descriptor back to Erlang over the Unix socket.

procket uses libancillary for passing file descriptors between processes:

    http://www.normalesup.org/~george/comp/libancillary/


## TODO

* Docs and type specs

* Try to re-use the Unix socket when requesting more fd's from the procket
  executable

* Make a procket gen\_server (gen\_raw(?)).
    * Support passive and active modes.
    * Hold state for the socket, so the caller does not need to, e.g.,
      use ifindex/2.
    * same interface for PF_PACKET and BPF


## CONTRIBUTORS

### Magnus Klaar
* support for binding a socket to an interface
* Makefile fixes

### Gregory Haskins
* fix link-error on SUSE platforms
