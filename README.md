LUKS TPM
========

A small utility script to manage LUKS keyfiles sealed by a v1.2 TPM.

This script assumes you will be using a TPM-sealed keyfile during boot to unlock
the root file system. It is intended to be used as part of your kernel update
process to generate a keyfile sealed against the new kernel's PCR values.

Requirements
------------

This script requires:

  * [bash](https://www.gnu.org/software/bash/)
  * [tpm-tools](http://sourceforge.net/projects/trousers)
  * [TrouSerS](http://sourceforge.net/projects/trousers)
  * [cryptsetup](https://gitlab.com/cryptsetup/cryptsetup)
  * A v1.2 TPM

Update Process
--------------

The script facilitates a variety of kernel update flows. For example, you could
set a temporary passphrase interactively during the update:

  1. Kernel is updated
  2. `luks-tpm temp` is called, either manually or via update hook, and sets
     a temporary LUKS passphrase
  3. The system is rebooted into the new kernel
  4. Because the TPM PCRs have changed, the old keyfile cannot be unsealed
  5. User enters the temporary passphrase to unlock the disk
  6. `luks-tpm reset` is called, generating a new keyfile sealed by the TPM and
     removing the temporary passphrase

### LUKS Key Slots

The script requires two LUKS key slots to function: one for the sealed keyfile
and one for the temporary passphrase. You are also *strongly* encouraged to
dedicate an additional slot for a recovery passphrase not managed by `luks-tpm`.

The default key slot layout is:

  * Slot 0: Recovery passphrase (optional)
  * Slot 1: TPM keyfile
  * Slot 2: Temporary passphrase

Usage
-----

    luks-tpm [OPTION]... [DEVICE] ACTION

### Actions

  * `init`: Initialize the LUKS TPM key slot
  * `temp`: Set a temporary LUKS passphrase
  * `reset`: Reset the LUKS TPM key using a passphrase
  * `replace`: Replace (overwrite) a LUKS TPM key

### Options

    -h         Print help
    -v         Print version information
    -m PATH    Mount point for the tmpfs file system used to store TPM keyfiles
               Default: /root/keyfs
    -k PATH    Sealed keyfile path
               Default: /boot/keyfile.enc
    -i HEX     Index of the TPM NVRAM area holding the key
    -s NUMBER  Key size in bytes
               Default: 32
    -t NUMBER  LUKS slot number for the TPM key
               Default: 1
    -r NUMBER  LUKS slot number for temporary reset passphrase
               Default: 2
    -p NUMBER  PCRs used to seal LUKS keyfile. May be specified more than once
               Default: 0-7
    -z         Use TSS well-known secret as owner password for NVRAM index usage
               or SRK password for keyfile sealing

### Actions

#### init

Initialize the LUKS TPM key slot, by default in LUKS slot 1. This action will
prompt for an existing LUKS passphrase and remove any existing key in slot 1.
It will then generate a random key, seal it with the TPM against the current
PCR values, and store the sealed key on disk or in NVRAM depending on the
options specified.

#### temp

Set a temporary LUKS passphrase. The TPM will be used to unseal the passphrase
for LUKS slot 1, which will be used to set a temporary passphrase in slot 2.
The user will be interactively prompted to enter this temporary passphrase.

#### reset

Prompts the user for the temporary passphrase (if needed) and uses it to set a
new passphrase in slot 1. The slot one key is then sealed by the TPM using the
current PCR values, and LUKS slot 2 is cleared.

#### replace

The `replace` action allows a TPM-sealed LUKS key to be replaced (overwritten)
by a new, randomly generated key. By default, LUKS slot 1 will be replaced.
This action will not prompt for a passphrase, so the current key must be both
"unsealable" by the TPM and a valid LUKS key.

### Default configuration

The script will read default configuration values by sourcing
`/etc/default/luks-tpm` if it exists. The location of this file can changed by
setting the `LUKSTPM_CONFIG` environment variable. Variables read from the
config file will override hard coded defaults, but will not override command
line arguments.

### LUKS device detection

If the path to a LUKS block device is not provided `luks-tpm` will use the
first device with a `crypto_LUKS` filesystem.

How-to
------

`luks-tpm` can protect LUKS keys using the TPM in one of two ways:

  * On disk as a "sealed" file that can only be decrypted by the TPM
  * In TPM non-volatile memory (NVRAM)

In either case, the data is only accessible when certain Platform Configuration
Registers (PCRs) have not changed. This indicates that the system has not been
altered since the data was sealed.

Before working with `luks-tpm`, ensure you can access the TPM on your system,
for example by running:

    $ sudo tpm_version

`luks-tpm` will attempt to guess the correct block device if it is omitted,
but it is recommended to specify the LUKS device. (/dev/sdaX in the examples
below)

## On-disk

Initialize `luks-tpm` with appropriate options:

    $ sudo luks-tpm -k /boot/keyfile.enc -H /dev/sdaX init

A sealed keyfile for LUKS slot 1 will be generated (in `/boot` for this
example): `/boot/keyfile.enc`.

The TSS well known secret can be used if you have not set a TPM owner password:

    $ sudo luks-tpm -z -k /boot/keyfile.enc -H /dev/sdaX init

## NVRAM

Most TPMs provide a small amount of user-configurable non-volatile memory
(NVRAM) that will perisist between reboots. Note that NVRAM often has a limited
number of writes, so it may not be a good option if frequent updates are
required.

Before initializing NVRAM storage, locate a free index:

    $ sudo tpm_nvinfo

And then call `luks-tpm` with appropriate options:

    $ sudo luks-tpm -i 0x100 /dev/sdaX init

The LUKS key for slot 1 will be stored in the TPM NVRAM at the specified
address,`0x100` in this example.

The TSS well known secret can be used if you have not set a TPM owner password:

    $ sudo luks-tpm -z -i 0x100 /dev/sdaX init

License and Copyright
---------------------

Copyright 2017-2020 Corey Hinshaw <corey@electrickite.org>

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
