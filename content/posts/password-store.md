+++
date = '2025-01-15T22:52:23Z'
draft = false
title = 'Password Store'
tags = ["linux"]
+++

## Prerequisites 
Being an `archlinux` user means dependencies are for `arch` but this blog post should work with any `nix` distribution.
- [Linux](https://wiki.archlinux.org/title/Installation_guide)

## What is pass?
`pass` is the standard Unix password manager. With `pass`, each password lives inside a `gpg` encrypted file where filename is the website label that requires password. These encrypted files are organised into meaningful folder hierarchies which can be copied from computer to computer and manipulated using standard command line.

I have an existing password store so the steps to set it up have extra steps to begin with.

## Import existing key
```
❯ gpg --import secret.gpg
gpg: key 35C5ACE1C3: public key "Don Manalo (lekat) <whoami@mailbox.com>" imported
gpg: key 35C5ACE1C3: secret key imported
gpg: Total number processed: 1
gpg:               imported: 1
gpg:       secret keys read: 1
gpg:   secret keys imported: 1
```

## Trust the imported key
```
❯ gpg --edit-key whoami@mailbox.com
gpg> trust
Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

gpg> save
```

## Initialise password store
```
❯ pass init whoami@mailbox.com
mkdir: created directory '/home/owner/.password-store/'
Password store initialized for whoami@mailbox.com
```

## Add password
```
 ❯ pass insert -m testpass
Enter contents of testpass and press Ctrl+D when finished:

THIS_IS_THE_PASSWORD
THIS_IS_THE_USERNAME
THIS_IS_THE_WEBSITE_URL
```

## Retrieve password to clipboard
```
❯ pass -c testpass
Copied testpass to clipboard. Will clear in 45 seconds.
```

## Display password
```
❯ pass show testpass
THIS_IS_THE_PASSWORD
THIS_IS_THE_USERNAME
THIS_IS_THE_WEBSITE_URL
```

## Deleted unwanted password
```
❯ pass rm testpass
Are you sure you would like to delete testpass? [y/N] y
removed '/home/drmanalo/.password-store/testpass.gpg'
```