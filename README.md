<div align="center">

# <img src="https://github.com/collabdt/docs/blob/main/static/img/cdt-logo.svg" alt="CDT Logo" height="40" />
#  `COLLAB DIGITAL TWINS`


**Democratizing Digital Twin Technologies**

</div>

##  DOCUMENTATION

<div align="center">
<em>
Communication is such an important aspect of good design. No matter how brilliant the product, if people cannot use it, it will receive poor reviews. It is up to the designer to provide the appropriate information to make the product understandable and usable.
</em>
<br>
— Don Norman, <em>The Design of Everyday Things</em>
</div>

This repository provides essential documentation to use and co-develop the CDT platform.
Don't hesitate to let us know if something is not clear and needs further description.
Write us at info@collabdt.org

# Website

This website is built using [Docusaurus](https://docusaurus.io/), a modern static website generator.

## Installation

```bash
yarn
```

## Local Development

```bash
yarn start
```

This command starts a local development server and opens up a browser window. Most changes are reflected live without having to restart the server.

## Build

```bash
yarn build
```

This command generates static content into the `build` directory and can be served using any static contents hosting service.

## Deployment

Using SSH:

```bash
USE_SSH=true yarn deploy
```

Not using SSH:

```bash
GIT_USER=<Your GitHub username> yarn deploy
```

If you are using GitHub pages for hosting, this command is a convenient way to build the website and push to the `gh-pages` branch.
