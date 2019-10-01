---
title: "Secrets in Git"
date: 2019-09-27T09:03:00+01:00
draft: false
---

In FWD we had an interesting problem, we had to build and release Android apps for many operators. All of them were built from the same source base using [Android flavors](https://developer.android.com/studio/build/build-variants). We started before Google Play offered the [managed signed certificates](https://developer.android.com/studio/publish/app-signing), hence we wanted a secure way to manage all the certificates, but also a way that could automate the build process. In these two posts I will try to explain the solution we implemented, using [git-crypt](https://www.agwa.name/projects/git-crypt/) and [abusing Gradle]({{<relref "abuse-android-builds">}}) a little.

There are different ways to use _git-crypt_. We used it with _gpg_ so we could have different users accessing the secrets without having just one single key. You can read the _git-crypt_ docs for other ways.

## GPG and git-crypt set up {#gpg}

To start you need to have git-crypt up and running in your machine (and in the others that will have access to the secrets). This instructions are for Mac, but I know that for Linux they are very similar.

```bash
$ brew install gpg2
$ brew install git-crypt
```

At the time that I did this installation, there was a small problem: git-crypt expects to have a `gpg` command in the system. But installing _gpg2_ you get a `gpg2` only. You will need to create a symlink in your `/usr/local/bin`. But this was some time ago, so check before you mess anything :)

```bash
lrwxr-xr-x  1 root    admin  27 Apr  5  2018 /usr/local/bin/gpg -> /usr/local/MacGPG2/bin/gpg2
```

If you have never used `gpg`, the first thing you need to do is to create a signing key:

```bash
$ gpg2 --gen-key
```

Now you need to init the repo.

```bash
$ cd repo
$ git-crypt init
```

It will create a `.gitattributes` file similar to:

```bash
secretfile filter=git-crypt diff=git-crypt
*.key filter=git-crypt diff=git-crypt
secretdir/** filter=git-crypt diff=git-crypt
```

You will need to modify the file. It follows the same idea as the `.gitignore`. You need to define which files will be encrypted. Be careful not to encrypt any files needed by git `.git*`. You need to edit this file before adding the sensitive files. Otherwise they will be in the history unencrypted. Adding them is as always, using `git add`.

To be able to access the files, the gpg keys need to be added to the ring. The first one is yourself, so you can see the encrypted files:

```bash
$ git-crypt add-gpg-user your@email.com
```

Make sure you add the `.gitattributes` and the new files to git, now you can commit. The files should be encrypted.

### Opening the secrets

If you have access to the files, opening the secrets is as easy as:

```bash
$ git-crypt unlock
```

And if you want to lock them again:

```bash
$ git-crypt lock
```

Easy!

### Adding more users

This is easy once you know what you need to do. It took me some time to figure out what was needed as I hadn't used _gpg_ before. To be able to give access to another user you need to receive the key from that person. The new person needs to:

```bash
$ gpg2 --gen-key  # only if they don't have one yet
$ gpg2 --export -o filename.key -a user2@email.com
```

The file generated is sent to a person who already has access to the secrets. And it needs to add the key to their _gpg_:

```bash
$ gpg2 --import filename.key
$ gpg2 --edit-key user2@email.com
```

In the edit mode you need to type: `trust`, `4`, `sign`, `save`. This is needed because the new key received first needs to be trusted by you. With those commands is what you do. Now you are set to add the person to _git_crypt_. Which is the same command that we used before when adding the first user:

```bash
$ git-crypt add-gpg-user user2@email.com
```

Commit the changes, and the new user should be able to `unlock` the secrets.

Now you should be ready to jump to the next part, [Abusing Android builds]({{<relref "abuse-android-builds">}}).