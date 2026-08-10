# Systemd integration

A Pingora server doesn't depend on systemd but it can easily be made into a systemd service.

```ini
[Service]
Type=forking
PIDFile=/run/pingora.pid
ExecStart=/bin/pingora -d -c /etc/pingora.conf
ExecReload=kill -QUIT $MAINPID
ExecReload=/bin/pingora -u -d -c /etc/pingora.conf
```

The example systemd setup integrates Pingora's graceful upgrade into systemd. To upgrade the pingora service, simply install a version of the binary and then call `systemctl reload pingora.service`.

## Socket activation

Graceful upgrade hands listening sockets from one Pingora process to the next, so the listeners survive an upgrade but not a plain restart. If the sockets should instead outlive every restart of the service, systemd can own them:

```ini
# pingora.socket
[Socket]
ListenStream=0.0.0.0:443
BindIPv6Only=both

# pingora.service
[Service]
ExecStart=/bin/pingora -c /etc/pingora.conf
```

With the socket unit active, systemd creates the listening sockets and passes them to the service through the `LISTEN_FDS` protocol. Clients connect to a socket that is never closed, so a restart drops no connections and triggers no reconnect storm.

Pingora does not read `LISTEN_FDS` itself; the application does, and hands the descriptors over before starting the server:

```rust
let mut fds = Fds::new();
// for each descriptor systemd passed, keyed by the address it is bound to
fds.add("0.0.0.0:443".to_string(), fd);

let mut server = Server::new(None)?;
server.set_listen_fds(fds);
server.bootstrap();
```

A listener whose address matches an entry is created from that descriptor instead of binding a fresh socket; anything unmatched binds normally, so a build that runs both under systemd and standalone needs no separate code path.

The keys must be exactly the strings the services bind. Any descriptor not claimed by a registered service is closed before startup, so registering one descriptor under several spellings of the same address closes it. Note that these strings are compared verbatim and are not canonicalized: `[::]:443` and `:::443` denote the same socket but are different keys.
