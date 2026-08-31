# Spandrel

This repository holds Spandrel's compiled, release-ready `spandrel.phar` —
the artifact `composer require spandrel/spandrel` installs, or that you
can download and run directly. It is published automatically from
[spandrel-php/spandrel-src](https://github.com/spandrel-php/spandrel-src),
where Spandrel's source, issues and pull requests live. See
[CHANGELOG.md](CHANGELOG.md) for what changed in each release.

## Usage

```sh
composer require --dev spandrel/spandrel
vendor/bin/spandrel analyse
```

Or run the PHAR directly:

```sh
curl -LO https://github.com/spandrel-php/spandrel/raw/0.1.2/spandrel.phar
php spandrel.phar analyse
```

Signed with `spandrel.phar.asc`. Verify against the bundled public
key (fingerprint `824682B3BE7DE81A8108FA98CE6235F34D7E5C43`):

```sh
gpg --import spandrel-bot.gpg.asc
gpg --verify spandrel.phar.asc spandrel.phar
```

The same key is also published on
[keys.openpgp.org](https://keys.openpgp.org), fetchable by
fingerprint without trusting this repo for the key itself:

```sh
gpg --keyserver hkps://keys.openpgp.org --recv-keys 824682B3BE7DE81A8108FA98CE6235F34D7E5C43
```
