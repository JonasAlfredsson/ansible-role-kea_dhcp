# ansible-role-kea_dhcp

This Ansible role will deploy up to three Docker containers, each running one
component of the [ISC Kea][8] DHCP server software and configure it to your
liking. The services available are:

- `kea-dhcp4`
- `kea-dhcp6`
- `kea-ctrl-agent`

> The containers used are [`jonasal/kea-<service>`][1] from [here][2] and use
> the `host` network [in order to function properly][7].



# Installation

This repository does not have any dependencies, so just move into your `roles/`
folder and run the following:

```bash
git clone git@github.com:JonasAlfredsson/ansible-role-kea_dhcp.git kea
```

If you would like to download any updates for this role in the future, you may
use the following command from within the previously cloned folder:

```bash
git pull
```

When the configuration is complete you may then just include this role in your
main playbook like this:

```yaml
- hosts: dhcp_servers
  name: Install Kea DHCP service
  roles:
    - kea
```



# Usage

Kea has quite a lot of configuration possibilities so it is impossible for me
to create a role that covers every conceivable usecase with easily accessible
variables, but I have tried to make the most common ones easy to achieve and
some more advanced stuff possible by just piping a
[custom variable](#raw-settings) through the `to_nice_json` filter in jinja.

I found it a little bit difficult to find a comprehensive list with all the
configuration options, but [here][3] is the documentation page which list
some things. However, I had to look at the example files found [here][4] to
actually get an understanding on how each setting should be defined.


## Example Configuration

To add some kind of structure to the example configuration I will split it into
five sections.

1. [Container Options](#container-options)
2. [DHCP Configuration](#dhcp-configuration)
    - [Subnets](#subnets)
3. [Control Agent Configuration](#control-agent-configuration)
4. [Logging](#logging)
5. [Raw Settings](#raw-settings)

All of the available settings can be viewed in the [`defaults`](./defaults/main.yml)
file, and it is recommended to check it there before you change anything since
I will only mention those I think are the most important here.


### Container Options
The settings grouped in the Container options section have more of an impact on
the host and the actual Docker container than the Kea service. One setting
that probably should be changed is the base directory in which the container
will find the configuration files and write its logs:

```yaml
kea_container_base_dir: "/srv/kea"
```

This is shared between all services, and they will use filenames that
differentiate between them instead. Something else that is important to note
here is that by default only the IPv4 service is enabled, and you will have to
change the rest if you want to include them:

```yaml
kea_dhcp4_enabled: true
kea_dhcp6_enabled: false
kea_ctrl_agent_enabled: false
```

### DHCP Configuration
These settings are targeted towards the DHCP services, and since these two have
very similar configuration files most options look like this:

```yaml
kea_dhcp_interfaces: [ '*' ]
kea_dhcp4_interfaces: "{{ kea_dhcp_interfaces }}"
kea_dhcp6_interfaces: "{{ kea_dhcp_interfaces }}"
```

which means that you either can change the setting "globally" or address
them individually per service. It is recommended that you explicitly name the
interface you want to listen to and don't use `*`, so this setting should most
likely be modified by you.

#### Subnets
One advanced variable available in this section is the `kea_dhcp*_subnets` one.
This is an empty list by default, but since this will be piped through the
`to_nice_json` filter in the jinja template it allows you to create a super
advanced configuration if you want. For example, here we create two subnets
with some extra options:

```yaml
kea_dhcp4_subnets:
  - subnet: "192.168.1.0/24"
    pools:
      - pool: "192.168.1.100 - 192.168.1.111"
      - pool: "192.168.1.200 - 192.168.1.222"
    reservations:
      - hw-address: "aa:bb:cc:dd:ee:fe"
        ip-address: "192.168.1.98"
      - hw-address: "aa:bb:cc:dd:ee:ff"
        ip-address: "192.168.1.99"
  - subnet: "192.168.2.0/24"
    hostname-char-replacement: "x"
    hostname-char-set: "[^A-Za-z0-9.-]"
    pools:
      - pool: "192.168.2.100 - 192.168.2.200"
    option-data:
      - always-send: false
        code: 3
        csv-format: true
```

There exists a plethora of settings here, so I suggest you look at the
"[all-keys][5]" example and build your own configuration from that.


### Control Agent Configuration
There are really only two settings here that are unique for the Control Agent,
and those are

```yaml
kea_ctrl_agent_host: "0.0.0.0"
kea_ctrl_agent_port: 8000
```

This will expose a RESTful API you can use to communicate with the other
services if they are up and running.


### Logging
Logging is important, else you do not know if anything goes wrong with the
services. You probably want the same settings across all services, so these
variables exist to make your life easier:

```yaml
# Note the space here----▼
kea_logging_pattern: "%D{ %Y-%m-%d %H:%M:%S.%q } %-5p [%c/%i.%t] %m\n"
kea_logging_severity: "INFO"
kea_logging_debuglevel: 0
kea_logging_maxsize: 10485760  # 10 MB
kea_logging_maxver: 3
```

Unless you change the `kea_logging_raw_*` variable this will set up logging at
the INFO level both to `stdout` and to a log file with automatic rotation when
it reaches 10MB in size and it will keep 2 files in history before they are
deleted. The only thing to be wary of here is the space in the `{ %` pointed out
above. This was necessary to do because `{%` is a special command in jinja which
makes the templating fail. Just be careful with that in case you want to change
the pattern.

However, like in the [subnet](#subnets) case discussed above we allow you to
pipe your own custom settings directly to the templating. Take a look in the
[defaults](./defaults/main.yml) file to see how the `kea_logging_raw_*`
variables are set up right now, and then at the [logging documentation][6] to
see if there is something you would like to change.

### Raw Settings
Finally there is the `kea_*_raw_options` variable. The same principle goes
here as in the [subnet](#subnets) and `kea_logging_raw_*` case where you can
just add freestyle options:

```yaml
kea_dhcp4_raw_options:
  authoritative: false
  option-def:
    - name: "vendor-encapsulated-options"
      code: 43
      type: "binary"
```

Only the sky and your sanity is the limit here, so go wild and use the
"[all-keys][5]" example as a source of inspiration.






[1]: https://hub.docker.com/search?q=jonasal%2Fkea
[2]: https://github.com/JonasAlfredsson/docker-kea
[3]: https://kea.readthedocs.io/en/latest/arm/config.html
[4]: https://github.com/isc-projects/kea/blob/master/doc/examples
[5]: https://github.com/isc-projects/kea/blob/master/doc/examples/kea4/all-keys.json
[6]: https://kea.readthedocs.io/en/latest/arm/logging.html
[7]: https://github.com/JonasAlfredsson/docker-kea#docker-network-mode
[8]: https://kea.readthedocs.io/en/latest/arm/intro.html
