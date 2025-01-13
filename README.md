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

Unclaimed ports: `5432`, `6379`, `9000`, `9001`, `3000`, `3001`

### Quick start

1. Clone this repository, i.e.: `git clone https://github.com/Ikigai-Systems/fundamento-standalone.git`
2. Run followoing command:
    ```
    docker compose run -ti --rm website -- bin/rails credentials:edit -e standalone 
    ```
    This will generate base secrets for your Fundamento instance and bring up Nano editor allowing you to edit them.
    Press Ctrl-X to leave the editor - you don't need to change anything there for your first Fundamento instance.
3. Build and start docker containers: `docker compose up`
4. Enjoy your Fundamento on: `http://localhost:3000`

### Optional - adjusting defaults

1. Fundamento will generate administrator user automatically when launching for the first time. You can use following
   credentials to log in:
   ```
   john@fundamento.it / secret!
   ```
   To change default administrator user you must edit `env.standalone` file but you need to do that **before** building
   Fundamento containers (step 3 from _Quick Start_ instructions).
2. Docker containers will be built using `latest` available version. If you need to use specific
   Fundamento's [version](https://github.com/Ikigai-Systems/fundamento-standalone/releases) adjust it in `docker-compose.yml` **before** building containers (step 3 from _Quick Start_ instructions).

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