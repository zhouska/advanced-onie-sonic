# Advanced ONIE DHCP examples for Dell Enterprise SONIC

Amazing things can be done with ONIE. Out with manual and error prone NOS instllation and in with 'everything is a code' approach. These examples are tested in real world scenarions and their aim is to save you precious time and repetitive lines of code.

# Prerequsites
1. ISC-DHCP server
2. Configure Vendor-Identifying Vendor-Specific Information (VIVSO)

## ISC-DHCP server

You can use provided docker file to buil your own ISC-DHCP server, or just download the `dhcpd.conf` and `hosts` files and tweak them to suit your needs.

Please note that the docker image uses `host` networking, so you need to have an interface with same subnet configured on the host that runs the container:

```
3: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:b4:5a:02 brd ff:ff:ff:ff:ff:ff
    inet 192.168.50.1/24 brd 192.168.50.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feb4:5a02/64 scope link
       valid_lft forever preferred_lft forever
```

You can then launch the docker image with the `docker-compose` command and provided `docker-compose.conf` file:

`docker-compose up -d`

## ISC-DHCP VIVSO configuration

We need to create ONIE namespace so we can parse the options and send responses back to requests made by individual instances.

```
# Create an option namespace called ONIE
option space onie code width 1 length width 1;

# Define the code names and data types within the ONIE namespace
option onie.installer_url code 1 = text;
option onie.updater_url   code 2 = text;
option onie.machine       code 3 = text;
option onie.arch          code 4 = text;
option onie.machine_rev   code 5 = text;

# Package the ONIE namespace into option 125
option space vivso code width 4 length width 1;
option vivso.onie code 42623 = encapsulate onie;
option vivso.iana code 0 = string;
option op125 code 125 = encapsulate vivso;

```
Source [ONIE VIVSO](https://opencomputeproject.github.io/onie/design-spec/discovery.html#vendor-identifying-vendor-specific-information-vivso)

## Install same ONIE image on a group of switch models (aka Match Vendor Class Identifier)

Since ISC-DHCP doesn't support wildcards `*` or regular expressions, we need to use `vendor-class-identifier` and tell it the length of the string we are looking for.

In this example, string `onie_vendor:x86_64-dellemc_n3248` is 32 bytes long and matches ALL N3200 platforms, such as N3248TE, N3248PXE, N3248X. Your can use this property to match other platfroms in same way. See the table [Dell Enterprise ONIE platforms](#dell-enterprise-onie-platforms) for details.

```
class "N3200" {
  # Limit the matching to a request we know originated from ONIE
  match if substring(option vendor-class-identifier, 0, 32) = "onie_vendor:x86_64-dellemc_n3248";

  # Required to use VIVSO
  option vivso.iana 01:01:01;

  # N series SONIC image
  option onie.installer_url = "http://192.168.50.1/onie-installer-n3200";

}

```
Source [ONIE Match Vendor Class Identifier](https://opencomputeproject.github.io/onie/user-guide/index.html#advanced-dhcp-match-vendor-class-identifier)

## Install same ONIE image on a given switch model (aka VIVSO)

In this example we pass same image to same switch model, each model can have a different image.

```
class "onie-vendor-classes" {
  # Limit the matching to a request we know originated from ONIE
  match if substring(option vendor-class-identifier, 0, 11) = "onie_vendor";

  # Required to use VIVSO
  option vivso.iana 01:01:01;

  # N3248PXE
  if option onie.machine = "dellemc_n3248pxe_c3338" {
    option onie.installer_url = "http://192.168.50.1/onie-installer-n3200";
  }

  # N3248TE
  if option onie.machine = "dellemc_n3248te_c3338" {
    option onie.installer_url = "http://192.168.50.1/sonic-4.1.bin";
  }

  # Z9100
  if option onie.machine = "dellemc_z9100_c2538" {
    option onie.installer_url = "http://192.168.50.1/onie-installer";
  }
}

```

Source [ONIE VIVSO](https://opencomputeproject.github.io/onie/user-guide/index.html#advanced-dhcp-2-vivso)

## Dell Enterprise SONIC ONIE platforms

You can use these ONIE platform identifiers in your DHCP configuration. Please note that you would need to remove the `x86_64-` string in order to be able to match the switch model in `onie.machine` DHCP option.

| Dell Technologies switch model | ONIE platform |
| :----------------------------- | :------------ |
| E3248P                         | x86_64-dell_e3248p-r0 |
| E3248PXE                       | x86_64-dell_e3248pxe-r0 |
| S6000                          | x86_64-dell_s6000_s1220-r0 |
| S6100                          | x86_64-dell_s6100_c2538-r0 |
| Z9100                          | x86_64-dell_z9100_c2538-r0 |
| N3248PXE                       | x86_64-dellemc_n3248pxe_c3338-r0 |
| N3248TE                        | x86_64-dellemc_n3248te_c3338-r0 |
| N3248X                         | x86_64-dellemc_n3248x_c3338-r0 |
| S5212F                         | x86_64-dellemc_s5212f_c3538-r0 |
| S5224F                         | x86_64-dellemc_s5224f_c3538-r0 |
| S5232F                         | x86_64-dellemc_s5232f_c3538-r0 | 
| S5248F                         | x86_64-dellemc_s5248f_c3538-r0 |
| S5296F                         | x86_64-dellemc_s5296f_c3538-r0 |
| Z9100                          | x86_64-dellemc_z9100_c2538-r0 |
| Z9264F                         | x86_64-dellemc_z9264f_c3538-r0 |
| Z9332F                         | x86_64-dellemc_z9332f_d1508-r0 |
| Z9432F                         | x86_64-dellemc_z9432f_c3758-r0 |
