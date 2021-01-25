### bmcbutler

[![Status](https://api.travis-ci.org/bmc-toolbox/bmcbutler.svg?branch=master)](https://travis-ci.org/bmc-toolbox/bmcbutler)
[![Go Report Card](https://goreportcard.com/badge/github.com/bmc-toolbox/bmcbutler)](https://goreportcard.com/report/github.com/bmc-toolbox/bmcbutler)
[![Development/Support](https://img.shields.io/badge/chat-on%20freenode-brightgreen.svg)](https://kiwiirc.com/client/irc.freenode.net/##bmc-toolbox)

##### About

Bmcbutler is a BMC (Baseboard Management Controller) configuration management tool that uses [bmclib](https://github.com/bmc-toolbox/bmclib).

## Configuration support

Hardware      | User accounts | Syslog  |  NTP  | Ldap  | Ldap groups  | BIOS  | HTTPS Cert  |
:-----------  | :-----------: | :-----: | :---: | :---: | :----------: | :--: | :---: |
Dell M1000e   | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | - | |
Dell iDRAC8   | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | | :heavy_check_mark: |
Dell iDRAC9   | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: |
HP c7000      | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | - | |
HP iLO4       | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | | :heavy_check_mark: |
HP iLO5       | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | | :heavy_check_mark: |
Supermicro X10 | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | :heavy_check_mark: | | :heavy_check_mark: |


Need help? See kiwiirc link above/find us on the freenode IRC channel `##bmc-toolbox`.

##### Build
`go get github.com/bmc-toolbox/bmcbutler`

###### Build with vendored modules (>= go 1.11)
`GO111MODULE=on go build -mod vendor -v`

###### Notes on working with go mod

To point to a local copy of bmclib, add to the bottom of the `go.mod` file

`replace github.com/bmc-toolbox/bmclib => ../bmclib`

To pick a specific bmclib SHA.

`GO111MODULE=on go get github.com/bmc-toolbox/bmclib@2d1bd1cb`

To add/update the vendor dir.

`GO111MODULE=on go mod vendor`

##### Setup
There's two parts to setting up configuration for bmcbutler,

* Bmcbutler configuration
* Configuration for BMCs

This document assumes the Bmcbutler configuration directory is ~/.bmcbutler.

###### Bmcbutler configuration
Setup configuration Bmcbutler requires to run.

```
# create a configuration directory for ~/.bmcbutler
mkdir ~/.bmcbutler/
```
Copy the sample config into ~/.bmcbutler/
[bmcbutler.yml sample](../master/samples/bmcbutler.yml)

###### BMC configuration
Configuration to be applied to BMCs.

```
# create a directory for BMC config
mkdir ~/.bmcbutler/cfg
```
add the BMC yaml config definitions in there, for sample config see [configuration.yml sample](../master/samples/cfg/configuration.yml)

###### bmc configuration templating
configuration.yml supports templating, for details see [configTemplating](../master/docs/configTemplating.md)

###### inventory
Bmcbutler was written with the intent of sourcing inventory assets and configuring their bmcs,
a csv inventory example is provided to play with.

[inventory.csv sample](../master/samples/inventory.csv.sample)

The 'inventory' parameter points Bmcbutler to the inventory source.

###### BMC HTTPS cert signing
Bmcbutler can manage certs for BMCs,
It compares the current HTTPS cert Subject attributes of a BMC with the ones declared in its configuration,
if the attributes don't match, it proceeds to,

1. Generate a CSR on the BMC using the Subject attributes declared in its configuration.
2. Pass the CSR to the signer executable, read the signed cert.
3. Upload the signed cert to the BMC.
4. Reset the BMC if required.

To have this setup,

1. Declare a `https_cert` configuration section in the BMC config template, see [configuration.yml sample](../master/samples/cfg/configuration.yml)
2. Declare a signer executable in the bmcbutler config, see [bmcbutler.yml sample](../master/samples/bmcbutler.yml)

The signer executable is required to accept a CSR through STDIN and spit out the signed cert through STDOUT.
An example signer that uses [lemur](https://github.com/Netflix/lemur) can be found under [helpers](../master/helpers)

###### Load credentials from [Vault](https://www.vaultproject.io)

Credentials to login to BMCs and configure them can be declared in the configuration file,
or can be looked up from Vault.

To setup secrets lookup from Vault,

- enable `secretsFromVault: true` in [bmcbutler.yml](../master/samples/bmcbutler.yml)
- Use the `lookup_secret::Administrator` parameter in place of the credential in [bmcbutler.yml](../master/samples/bmcbutler.yml)
- Use the `<%= lookup_secret("Administrator") %>` YAML templating parameter in place of credentials in [configuration.yml sample](../master/samples/cfg/configuration.yml)
- See the sample bmcbutler.yml for options to set the vault token.

Examples

Set credentials in Vault, using `--config` and command substitution to prevent leaking the vault token
to other processes (command line arguments are visible to all processes).
```
curl --config <( builtin printf 'header = "X-Vault-Token: %s"' "${VAULT_TOKEN}" ) \
    -H "Content-Type: application/json" \
    -X POST -d '{"Administrator": "hunter2", "Ops": "foobar"}' https://vault.example.com/v1/secret/baremetal/bmc
```

Check credentials were set
```
curl --config <( builtin printf 'header = "X-Vault-Token: %s"' "${VAULT_TOKEN}" ) \
      -X GET https://vault.example.com/v1/secret/baremetal/bmc
```

bmcbutler.yml - declare Vault config and replace credentials
```
secretsFromVault: true
vault:
  hostAddress: "http://172.18.0.2:8200"
  tokenFromFile: "samples/vault-token.test"
  secretsPath: /secret/baremetal/bmc
credentials:
  - Administrator: lookup_secret::Administrator
  - Administrator: lookup_secret::Admin2
  - root: lookup_secret::dell_default
  - ADMIN: lookup_secret::sm_default
```

configuration.yml - declare BMC user account config with `lookup_secrets` template method.
```
user:
  - name: Administrator
    # lookup_secret - requires 'secretsFromVault: true' in bmcbutler.yml
    # note - double quotes required
    password: <%= lookup_secret("Administrator") %>
    role: admin
    enable: true
  - name: Ops
    password: <%= lookup_secret("Ops") %>
    role: user
    enable: false
```

##### Run

Configure Blades/Chassis/Discretes

```
#configure all BMCs in inventory, dry run with debug output
bmcbutler configure --all --dryrun --debug

#configure all servers in given locations
bmcbutler configure --servers --locations ams2

#configure all chassis in given locations
bmcbutler configure --chassis --locations ams2,lhr3

#configure all servers in given location, spawning given butlers
bmcbutler configure --servers --locations lhr5 --butlers 200

#configure one or more BMCs identified by IP(s)
bmcbutler configure --ips 192.168.0.1,192.168.0.2,192.168.0.2

#configure one or more BMCs identified by serial(s) and trace log
bmcbutler configure --serials <serial1>,<serial2> --trace

bmcbutler configure --serial <serial1>,<serial2> --debug
bmcbutler configure  --serial <serial> --debug

#Apply specific configuration resource(s) and trace log
bmcbutler configure --ips 192.168.1.4 --resources ntp,syslog,user --trace
```

#### Acknowledgment

bmcbutler was originally developed for [Booking.com](http://www.booking.com).
With approval from [Booking.com](http://www.booking.com), the code and
specification were generalized and published as Open Source on github, for
which the authors would like to express their gratitude.
