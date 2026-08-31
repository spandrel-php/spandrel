# Changelog

## 0.2.2 - 2026-08-31

- Fix analyse silently falling back to stdout on an unwritable report target (#15) ([94389f8](https://github.com/spandrel-php/spandrel-src/commit/94389f8cd290a43565bb9f67da4e87a411eb6f00))
- Add explicit least-privilege permissions to create-tag.yml (#16) ([aa4580f](https://github.com/spandrel-php/spandrel-src/commit/aa4580fbbabe446a80243462a868136aec28df03))
- Harden the build/phar jobs: verified cache, no install-time scripts, audit gate (#17) ([07f3375](https://github.com/spandrel-php/spandrel-src/commit/07f3375cffe7369403fe730abcde66074a166a51))
- Harden release.yml publish job against untrusted-input shell shapes (#18) ([26df0da](https://github.com/spandrel-php/spandrel-src/commit/26df0dabfd896fafed75e5a8c8e7e644573d8a63))
- Close the unverified path in phar verification instructions (#19) ([c70b192](https://github.com/spandrel-php/spandrel-src/commit/c70b19276d4092f7fe0910b91f5a45f6875d4bcb))

## 0.2.1 - 2026-08-31

- Fix release signing failing on the second+ release (#5) ([0092705](https://github.com/spandrel-php/spandrel-src/commit/009270580b839e376759e9bc1f7b98c7717018cf))
- Trim workflow comments down to durable why, drop bug narration (#6) ([86f40d4](https://github.com/spandrel-php/spandrel-src/commit/86f40d44ec57781e34f31bc0c4301ef7d60afc57))

## 0.1.2 - 2026-08-31

- Fix Commit and tag step failing against the still-empty dist repo ([f5e3eae](https://github.com/spandrel-php/spandrel-src/commit/f5e3eae9e87b695f995bcae2aa454f15bb345fec))
