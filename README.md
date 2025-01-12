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

## Getting started

### Requirements

Unclaimed ports: `5432`, `6379`, `9000`, `9001`, `3000`, `3001`

### Installation

1. Clone this repository, i.e.: `git clone https://github.com/Ikigai-Systems/fundamento-standalone.git`
2. Adjust environment variables:
   1. `cp env.sample env.standalone`
   2. edit `env.standalone` and configure account for the first user
3. Optionally adjust Fundamento's [version](https://github.com/Ikigai-Systems/fundamento-standalone/releases) in `docker-compose.yml` - otherwise `latest` will be used
4. Initialize secrets with the following command:
    ```
    docker compose run -ti --rm website -- bin/rails credentials:edit -e standalone 
    ```
    In most cases you don't need to change anything in this file.
5. Build and start docker containers: `docker compose up`
6. Enjoy your Fundamento on: `http://localhost:3000`

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