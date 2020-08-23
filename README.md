## Postfix rate limit policy delegation service using Nginx

This project is an experiment to try to build a [Postfix Policy Delegation](http://www.postfix.org/SMTPD_POLICY_README.html) service based on Nginx.

Postfix Policy Delegation is quite simple. It sends a number of `name=value` attributes, and expects an `action=value` reply. This is all done over TCP or unix domain sockets.

There are [many](http://www.postfix.org/addon.html) Policy Delegation servers available. Most of them are programmed using C, perl, python, or any other languages. The idea here is to focus on the service logic and to leave everything else (access control, resource management, load balancing, ...) to Nginx. 

Nginx is not only a HTTP server. It also features a Stream Module that fits perfectly well for this use case. The [core Nginx version](http://nginx.org/en/docs/stream/ngx_stream_core_module.html) is mostly used as a proxy to external services. We'll be using the [Lua enabled version](https://github.com/openresty/stream-lua-nginx-module) provided by [Openresty](https://openresty.org/en/). This will allow us to embed the logic in Nginx.


### Rate limit

The use case for this project is a rate limit service. Nginx will read data from Postfix and return either "dunno" or "defer", depending on the configured limits, for any of these Postfix Policy Delegation protocol attributes:

- helo_name
- sender
- recipient
- client_address
- client_name
- reverse_client_name
- sasl_username
- sasl_sender

The application allows counters to be stored either locally or in [Weakforced](https://github.com/PowerDNS/weakforced). The latter is the key to clustered setups where many Nginx instances share rate limiting data.

For the rate limit algorithm, it's quite common to use either sampling periods or leaky buckets. I wanted to test this [sliding window implementation](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/), so this is what I'm using.

## Installation

You need a Nginx package that includes both the stream module and Lua. Use whatever you prefer, but I would recommend [Openresty](http://openresty.org/en/).

The only real Lua dependency for this project is the Lua JSON module. If you choose Openresty, it's already there. If you don't, it's usually easy to install.

If you want many Nginx and Postfix instances to share counters, install Weakforced. If you don't know what it is, take your time, as it is a fantastic piece of software you should definitively know about.

### Nginx

After Nginx is installed, make sure to copy the sample `nginx.conf` stream module configuration. Feel free to change the default `/usr/local/etc/policy.conf` path at `init_worker_by_lua_block`. Nginx reads this file every minute to update configured limits.

You can also change the server `listen` port. This is what postfix uses to connect to the service.

### Postfix

`smtpd_recipient_restrictions` is probably where you want to use the Policy Service. For example:

```
smtpd_recipient_restrictions =
  ...
  check_policy_service { inet:127.0.0.1:10000, timeout=5s, default_action=DUNNO },
  ...
```

### Weakforced

Weakforced is not needed for single node installations, where counters can be stored locally. The Policy Service uses a custom Weakforced endpoint and a database. Make sure that the sample `wforce.conf` is merged in your configuration. Remember that the sample configuration defines a database to store two 10 minute buckets.

## Configuration

The Policy Service reads configuration data from `/usr/local/etc/policy.conf`. This is currently hardcoded. Nginx reloads the file every minute.

```
{
  "wforce_user": "wforce_u",
  "wforce_pass": "wforce_p",
  "wforce_host": "127.0.0.1",
  "wforce_port": "8084",
  "default_local_sender": 10,
  "default_cluster_sender": 30,
  "sender": [
              {"test1@example.com": 3},
              {"test2@example.com": 5}
          ],
  "client_address": [
              {"127.0.10.1": 10},
              {"127.0.10.2": 11}
          ],
  "helo_name": [
              {"mailserver01.example.com": 4}
          ]
}
```

If any of the wforce_* parameters is not configured, the service falls back to local mode, with counters stored locally. 

`default_local_*` and `default_cluster_*` define the default limits for the available Postfix Policy Delegation protocol attributes (sender, client_address, ...).

You can also define limits for specific IPs, hostnames, addresses, .... At this moment, these limits are applied to both local and cluster modes.

The configuration file is read every minute, so you should not remove entries from the file unless you want to remove the limits.

## TODO

This is a very simple test. A production ready implementation should be more dynamic (rates, buckets, ...), include regular expressions, ....