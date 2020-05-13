# JOB Multi ID Provider Auth Page

See [documentation](TBD).

## Definition

A demo instance of a **JOB Auth** web page (JOBA) is available at https://jupyter-observablehq-bridge.github.io/multi-id-auth-page.

It allows a user to authenticate with a number of ID providers:

- [Google single signin](https://developers.google.com/identity/sign-in/web/sign-in)
- SG Connect (corporate example)

To add/remove other ID providers simply clone the repo and publish your own JOBA.

Most main stream ID providers offer off-the-shelf JavaScript libraries that encapsulate the complexity of the OAuth 2.0 mechanism (redirection, refresh, etc), are typically imported via top level `<script>` tags, and then simple to use.

Consequently a JOBA can be a simple aggregate of several ID providers user friendly authentication libraries.  
To make that even simpler the demo JOBA is written in [Vue](https://vuejs.org/) for structure but does **not** require a build step.  
So it is easy to fork and adapt.

## Dev

- launch web servers:

```bash
python -m http.server 3000
python -m http.server 3001
```

- Open http://localhost:3001/opener.html
- Click button "Open Auth Page..." and browser console
- This will open page http://localhost:3000
- Login to Google and/or SG Connect
- See http://localhost:3001/opener.html console

## Deploy

- Install [push-dir](https://www.npmjs.com/package/push-dir)

```bash
yarn add push-dir
```

- push `master` branch to `gh-pages`.

```bash
push-dir --dir=. --branch=gh-pages
```
