# tiller-circleci

Deploy Trellis, Bedrock and Sage via CircleCI.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Requirements](#requirements)
- [What's in the Box?](#whats-in-the-box)
- [File Structures](#file-structures)
  - [Official](#official)
  - [Typist Tech](#typist-tech)
- [SSH Key](#ssh-key)
  - [GitHub](#github)
  - [CircleCI](#circleci)
  - [Trellis](#trellis)
- [Ensure Trellis Deploys the Correct Commit](#ensure-trellis-deploys-the-correct-commit)
- [Ansible Vault Password](#ansible-vault-password)
- [FAQ](#faq)
  - [Is it a must to merge Trellis pull request #997?](#is-it-a-must-to-merge-trellis-pull-request-997)
  - [What is in the `itinerisltd/tiller` docker image?](#what-is-in-the-itinerisltdtiller-docker-image)
  - [Is it a must to use all Trellis, Bedrock and Sage?](#is-it-a-must-to-use-all-trellis-bedrock-and-sage)
  - [Is it a must to use CircleCI?](#is-it-a-must-to-use-circleci)
  - [Is it a must to use GitHub?](#is-it-a-must-to-use-github)
  - [What does it cache?](#what-does-it-cache)
  - [It looks awesome. Where can I find some more goodies like this?](#it-looks-awesome-where-can-i-find-some-more-goodies-like-this)
  - [This package isn't on wp.org. Where can I give a :star::star::star::star::star: review?](#this-package-isnt-on-wporg-where-can-i-give-a-starstarstarstarstar-review)
- [Support](#support)
  - [Why don't you hire me?](#why-dont-you-hire-me)
  - [Want to help in other way? Want to be a sponsor?](#want-to-help-in-other-way-want-to-be-a-sponsor)
- [Author Information](#author-information)
- [Feedback](#feedback)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Requirements

- [Trellis](https://github.com/roots/trellis) with pull request [#997](https://github.com/roots/trellis/pull/997) merged
- [CircleCI](https://circleci.com)
- (Optional) Bedrock [31c638f](https://github.com/roots/bedrock/commit/31c638fe5a33d9e40a5c9eee07a94deb9cb76adc) or later
- (Optional) Sage [9.0.1](https://github.com/roots/sage/releases/tag/9.0.1) or later

## What's in the Box?

`.circleci/config.yml` examples of running [Trellis deploys](https://roots.io/trellis/docs/deploys/) to production whenever master branch is pushed.

## File Structures

Tiller comes with 2 different `config.yml` examples. They are expecting different Trellis and Bedrock structures.

### Official

Use [`config.yml`](./config.yml) if your directory structure follow [the official documents](https://roots.io/trellis/docs/installing-trellis/#create-a-project):
```
example.com/      # → Root folder for the project
├── .git/         # → Only one git repo
├── trellis/      # → Your clone of roots/trellis, directory name must be `trellis`
└── site/         # → A Bedrock-based WordPress site, directory name doesn't matter
```

To install `config.yml`:
1. Set up SSH keys, Ansible Vault password and commit Trellis changes described in the following sections
1. Copy, review, change and commit [`config.yml`](./config.yml) to `.circleci/config.yml`

### Typist Tech

At [Typist Tech](https://typist.tech/), I use a opinionated project structure:
- separate Trellis and Bedrock as 2 different git repo
- name the Bedrock-based WordPress site directory more creatively, i.e: `bedrock`

```
example.com/      # → Root folder for the project
├── bedrock/      # → A Bedrock-based WordPress site, directory name doesn't matter
│   └── .git/     # Bedrock git repo
└── trellis/      # → Clone of roots/trellis, directory name must be `trellis`
    └── .git/     # Trellis git repo
```

See: [roots/trellis#883 (comment)](https://github.com/roots/trellis/issues/883#issuecomment-329052189)

To install `config.typisttech.yml`:
1. Set up SSH keys, Ansible Vault password and commit Trellis changes described in the following sections
1. Push the Trellis repo
1. Copy, review, change and commit [`config.typisttech.yml`](./config.typisttech.yml) to `<bedrock>/.circleci/config.yml`

---

## SSH Key

You need a robot user for deployment. In this example, we will use a GitHub machine user account as our robot. For simplicity, this robot uses the same SSH key pair to access both GitHub private repos and the web server.

### GitHub

1. Sign up a machine user(e.g: `mybot`) on GitHub
1. Grant `mybot` **read** access to all necessary private repos

### CircleCI

On CircleCI's web console:
1. Link your project repo
1. Go to **Settings** » **Checkout SSH Keys**
1. Delete the deploy key
1. Create a user key (as `mybot`)

Learn more about deploy keys and user keys on CircleCI **Checkout SSH Keys** settings page.

### Trellis

1. Add the SSH key to web server
    ```diff
    # group_vars/<env>/users.yml
    users:
      - name: "{{ web_user }}"
        groups:
          - "{{ web_group }}"
        keys:
          - https://github.com/human.keys
    +      - https://github.com/mybot.keys
      - name: "{{ admin_user }}"
        groups:
          - sudo
        keys:
          - https://github.com/human.keys
    ```
1. Re-provision
    `$ ansible-playbook server.yml -e env=<env> --tags users`

## Ensure Trellis Deploys the Correct Commit

Normally, Trellis always deploy the **latest** commit of the branch. We need a change in `group_vars/<env>/wordpress_sites.yml`:

```diff
# group_vars/<env>/wordpress_sites.yml
wordpress_sites:
  example.com:
-    branch: master
+    branch: "{{ site_version | default('master') }}"
```

## Ansible Vault Password

Unlike other environment variables, [Ansible Vault](https://docs.ansible.com/ansible/playbooks_vault.html) password should never be stored as plaintext. Therefore, you should add `VAULT_PASS` via [CircleCI web console](https://circleci.com/docs/2.0/env-vars/#setting-an-environment-variable-in-a-project) instead of commit it to `.circleci/config.yml`.

The examples assume you have defined `vault_password_file = .vault_pass` in `ansible.cfg` as [the official document](https://roots.io/trellis/docs/vault/#2-inform-ansible-of-vault-password) suggested.

```diff
# ansible.cfg
[defaults]
+ vault_password_file = .vault_pass
```

To use another vault password filename:
```diff
- run:
    name: Set Ansible Vault Pass
-    command: echo $VAULT_PASS > .vault_pass
+    command: echo $VAULT_PASS > .my_vault_password_file
    working_directory: trellis
``

Using [Ansible Vault](https://docs.ansible.com/ansible/playbooks_vault.html) to encrypt sensitive data is strongly recommended. In case you have a very strong reason not to use Ansible Vault, remove the step:

```diff
-- run:
-    name: Set Ansible Vault Pass
-    command: echo $VAULT_PASS > .vault_pass
-    working_directory: trellis
```

## FAQ

### Is it a must to merge Trellis pull request [#997](https://github.com/roots/trellis/pull/997)?

Yes.

### What is in the `itinerisltd/tiller` docker image?

It is maintained by the [Tiller](https://github.com/ItinerisLtd/tiller) project.
Read its [readme](https://github.com/ItinerisLtd/tiller#docker-image-1) to learn more.

### Is it a must to use all Trellis, Bedrock and Sage?

No, you don't need all of them. Only Trellis is required.

### Is it a must to use CircleCI?

No. The original [Tiller](https://github.com/ItinerisLtd/tiller) project uses AWS CodeBuild. You can tweak it to run on different CI providers.

### Is it a must to use GitHub?

No. GitHub is just an example.

### What does it cache?

By default, yarn packages, Ansible Galaxy roles and [Trellis' temporary build directory](https://github.com/roots/trellis/pull/997) are cached. It speeds up the build significantly.
This is optional and you can [customize the cache behaviour](https://circleci.com/docs/2.0/caching/).

### It looks awesome. Where can I find some more goodies like this?

* Articles on Typist Tech's [blog](https://typist.tech)
* [Tang Rufus' WordPress plugins](https://profiles.wordpress.org/tangrufus#content-plugins) on wp.org
* More projects on [Typist Tech's GitHub profile](https://github.com/TypistTech)
* Stay tuned on [Typist Tech's newsletter](https://typist.tech/go/newsletter)
* Follow [Tang Rufus' Twitter account](https://twitter.com/TangRufus)
* Hire [Tang Rufus](https://typist.tech/contact) to build your next awesome site

### This package isn't on wp.org. Where can I give a :star::star::star::star::star: review?

Thanks!

Consider writing a blog post, submitting pull requests, [donating](https://typist.tech/donation/) or [hiring me](https://typist.tech/contact/) instead.

## Support

Love `tiller-circleci`? Help me maintain it, a [donation here](https://typist.tech/donation/) can help with it.

### Why don't you hire me?

Ready to take freelance WordPress jobs. Contact me via the contact form [here](https://typist.tech/contact/) or, via email [info@typist.tech](mailto:info@typist.tech)

### Want to help in other way? Want to be a sponsor?

Contact: [Tang Rufus](mailto:tangrufus@gmail.com)

## Author Information

[tiller-circleci](https://github.com/typisttech/tiller-circleci) is a [Typist Tech](https://typist.tech/) project created by [Tang Rufus](https://typist.tech).

Special thanks to [Itineris Limited](https://www.itineris.co.uk/) who hired me to create the original [Tiller](https://github.com/ItinerisLtd/tiller) project.

Special thanks to [the Roots team](https://roots.io/about/) whose [Trellis](https://github.com/roots/trellis) make this project possible.

Full list of contributors can be found [here](https://github.com/ItinerisLtd/tiller/graphs/contributors).

## Feedback

**Please provide feedback!** We want to make this library useful in as many projects as possible.
Please submit an [issue](https://github.com/ItinerisLtd/tiller/issues/new) and point out what you do and don't like, or fork the project and make suggestions.
**No issue is too small.**
