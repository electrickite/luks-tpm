LUKS TPM
========

A small utility script to manage LUKS keyfiles sealed by a TPM.

This script assumes you will be using a TPM-sealed keyfile during boot to unlock
the root file system. It is intended to be used as part of your kernel update
process to generate a keyfile sealed against the new kernel's PCR values.

Update Process
--------------

The script facilitates the following kernel update process:

  1. Kernel is updated
  2. `luks-tpm temp` is called, either manually or via pacman hook, and sets
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

### Replace Key

The `replace` action allows a TPM-sealed LUKS keyfile to be replaced
(overwritten) by a new, randomly generated key. By default, LUKS slot 1 will be
replaced. This action will not prompt for a passphrase, so the current keyfile
must "unsealable" by the TPM and a valid LUKS key.

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
    -p PATH    Sealed keyfile path
               Default: /boot/keyfile.enc
    -x HEX     Index of the TPM NVRAM area holding the key
    -s NUMBER  Key size in bytes
               Default: 32
    -t NUMBER  LUKS slot number for the TPM key
               Default: 1
    -r NUMBER  LUKS slot number for temporary reset passphrase
               Default: 2
    -L NUMBER  PCRs used to seal LUKS keyfile. May be specified more than once
               Default: 0-7
    -z         Use TSS well-known secret as owner password for NVRAM index usage
               or SRK password for keyfile sealing
               (insecure, use for demonstrative purposes only)
