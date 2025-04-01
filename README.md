# Fundamento - Strong foundation for your internal collaboration

<p align="center">
  <a href="https://fundamento.it" target="_blank" align="center">
    <img src="https://res.cloudinary.com/fundamento/image/upload/v1734469016/fundamento_banner.webp" width="900" alt="Fundamento Banner">
  </a>
  <br>
</p>

---

## Features

* One
* Two
* Three

---

## Installation

### Requirements

Unclaimed port: `3000`

### Quick start

1. Clone this repository, i.e.: `git clone https://github.com/Ikigai-Systems/fundamento-standalone.git`
2. By default, Fundamento will create superadmin user for you with the following credentials to log in:
   ```
   john@fundamento.it / secret!
   ```
   If you plan to only play with Fundamento it's ok to leave it as is. For production sites you should
   configure unique superadmin user though. To do that, you need to edit `env.standalone` file before next steps.
3. Generate base secrets for your Fundamento instance:
   ```
   docker compose run -ti --rm website -- bin/rails credentials:edit -e standalone
   ```
   This will bring up Nano editor allowing you to adjust generated values. Press Ctrl-X to leave the editor -
   you don't need to change anything for your first Fundamento instance.
4. Build and start docker containers: `docker compose up`
5. Enjoy your Fundamento on: `http://localhost:3000`

### Selecting version

Docker will download and cache the `latest` available Fundamento version. If you need to use specific
Fundamento's [version](https://github.com/Ikigai-Systems/fundamento-standalone/releases), or want to install Fundamento on host used for installation in the past, you
need to adjust version in `docker-compose.yml` **before** building containers (step 3 from _Quick Start_ instructions).

### Troubleshooting

_\<to be filled in later with real examples we'd encounter\>_

---

## Documentation

* https://docs.fundamento.it

---

## Development

### Making a release

* Create a release in https://github.com/Ikigai-Systems/fundamento-cloud
* Use the same release name in this repository \
  Example:
  ```
  gh release create -R ikigai-systems/fundamento-cloud v0.0.1-test.4
  gh release create -R ikigai-systems/fundamento-standalone v0.0.1-test.4
  ```
* Official release images will be created in this repository.

---

<p align="center">
<a href="https://fundamento.it">Fundamento</a> &bull;
<a href="https://docs.fundamento.it">Docs</a> &bull;
<a href="https://fundamento.it/pricing">Pricing</a> &bull;
<a href="https://fundamento.it/terms">Terms</a> &bull;
<a href="https://fundamento.it/privacy">Privacy</a>
</p>