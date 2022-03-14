# Yubikey x GnuPG

Using GNU Pretty Good Privacy (PGP) with YubiKey

## Tools

    brew install gpg

## Creating Keys

If you do not have an existing GPG keychain, you will need to create one.
You can follow [this guide](https://wiki.archlinux.org/index.php/GnuPG#Create_a_key_pair).

Some quick snippets:

    gpg --expert --full-gen-key
    xx # (9) ECC and ECC
    xx # (1) Curve 25519
    (this should create a secret key and also an encryption key)

## Listing Keys

To list keys in your public key ring:
    
    gpg --list-keys

To list keys in your secret key ring: 
    
    gpg --list-secret-keys
    
Note the key ID which will be used in the previous step.
The ID starts after the "/". Given `rsa4096/34393DA618A60674`, the key ID is `34393DA618A60674`.
    
### Symbols and etc

- `pub` - public key
- `sub` - subkey
- `sec` - private (or secret) key
- `ssb` - private (or secret) subkey
- `uid` - user id
- `...>`: key stored on smart card
- `...#`: key is unusable - likely secret key missing

## Edit GPG database

    gpg --edit-key <uid>
    
The UID is in the form of an email (usually email of the owner).

## Revoking a subkey

    gpg --edit-key <uid>
    key <key_id>
    revkey
    (follow the instruction on screen)
    
## Adding a new UID or email

    gpg --edit-key <uid>
    adduid
    (follow the instruction on screen)
    trust
    (follow the instruction on screen)
    
## Add a new subkey

    gpg --edit-key <uid>
    addkey
   
## Show the `key_id` and keygrip

    gpg --edit-key <uid>
    grip
    
The keygrip could correspond to a file inside `.gnupg/private-keys-v1.d` directory.

## Add a new authentication subkey

    gpg --edit-key <uid>
    addkey
    (note that there isn't an option that allow set your own capabilities)
    ctrl+D
    
Let's start again.

    gpg --expert --edit-key <uid>
    addkey
    xx # ECC (set your own capabilities)
    toggle to remove sign capability
    toggle to add authenticate capability
    xx # (1) Curve 25519
    save
    
## Changing PIN for card

For more info: https://developers.yubico.com/PGP/Card_edit.html

    gpg --card-edit
      
  
## Moving key to card

Ensure that your YubiKey is plugged in and your PIN change.

    gpg --edit-key <uid>
    key <key id>
    (note the * beside the ssb)
    keytocard
    save
    gpg --card-edit
    (you should see the key you just move show up in the right slot)
    quit
    
## Using Yubikey only

Generate the public key chain:

    gpg --armor --export > <some path>

Removing the `.gnupg` folder (which I typically store in Dropbox).
Then import the public key chain:

    gpg --import <the path you put above>
    (should say the key is imported)
    gpg --list-keys
    (should say the public key that you exported as previously)
    gpg --list-secret-keys --with-subkey-fingerprint
    (should print out nothing)
    gpg --card-edit
    fetch
    (to install the key)
    gpg --edit-key <uid>
    trust
    5 (I trust ultimately)
    gpg --edit-key <uid>
    (should see the trust change to ultimate)
    gpg --card-status
    (should show the ssb> for the right keys that you move to the card)
    
## Exporting SSH public key

    gpg --export-ssh-key <uid>
    
## Changing expiry date of a key

    gpg --expert --edit-key <uid>
    key X
    expire

## Setup your MacOS environment to use GPG key for SSH authentication
    
Create the files as follows:
    
    # ~/.gnupg/gpg-agent.conf
    enable-ssh-support
    pinentry-program /usr/local/bin/pinentry-mac
    
    # ~/.config/fish/config.fish
    set -e SSH_AUTH_SOCK
    set -U -x SSH_AUTH_SOCK (gpgconf --list-dirs agent-ssh-socket)
    set -x GPG_TTY (tty)
    gpgconf --launch gpg-agent
    gpg-connect-agent updatestartuptty /bye > /dev/null
    
## Signing your git commit with GPG 

    gpg --list-secret-keys --keyid-format LONG
    (get the signing key id)
    
Update the following lines inside `~/.gitconfig`:

    [user]
    	signingkey = <key id>
    [commit]
    	gpgsign = true

Test a sample commit.

You can check the signature via:

    git log --show-signature -1
    git show HEAD --show-signature

## Exporting your signature and tell other provider (e.g. GitHub)

    gpg --armor --export <key id>
    (copy from -----BEGIN PGP PUBLIC KEY BLOCK----- to -----END PGP PUBLIC KEY BLOCK-----)

## What to do when your commit signing key expires

1. Switch to the `.gnupg` keychain where the secret key exists
1. [Add a new signing key](#add-a-new-subkey)
1. [Move the new key to the card](#moving-key-to-card)
1. Revoke the signature

        gpg --expert --edit-key <uid>
        key X
        revsig
1. Export the public key chain for use in the later steps

        gpg --armor --export > <some path>

1. Switch to the `.gnupg` without the secret key
1. [Import from Yubikey](#using-yubikey-only)


## Notes

If you encounter the following in `ssh -vv`:

    sign_and_send_pubkey: signing failed: agent refused operation
    
Run 
    
    gpg-connect-agent updatestartuptty /bye.
    
    
## Reference

- https://developers.yubico.com/PGP/Importing_keys.html
- https://developers.yubico.com/PGP/Card_edit.html
- https://help.github.com/en/github/authenticating-to-github/checking-for-existing-gpg-keys
- https://blog.summercat.com/using-a-yubikey-for-ssh-authentication.html
- https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work