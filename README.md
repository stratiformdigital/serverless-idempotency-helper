<h1 align="center" style="border-bottom: none;"> serverless-idempotency-helper</h1>
<h3 align="center">Make lambda deployments more idempotent.</h3>
<p align="center">
  <a href="https://github.com/stratiformdigital/serverless-idempotency-helper/releases/latest">
    <img alt="latest release" src="https://img.shields.io/github/release/stratiformdigital/serverless-idempotency-helper.svg">
  </a>
  <a href="https://www.npmjs.com/package/@stratiformdigital/serverless-idempotency-helper">
    <img alt="npm latest version" src="https://img.shields.io/npm/v/@stratiformdigital/serverless-idempotency-helper/latest.svg">
  </a>
  <a href="https://codeclimate.com/github/stratiformdigital/serverless-idempotency-helper/maintainability">
    <img alt="Maintainability" src="https://api.codeclimate.com/v1/badges/35345bf3fab6e09eaefa/maintainability">
  </a>
  <a href="https://github.com/semantic-release/semantic-release">
    <img alt="semantic-release: angular" src="https://img.shields.io/badge/semantic--release-angular-e10079?logo=semantic-release">
  </a>
  <a href="https://dependabot.com/">
    <img alt="Dependabot" src="https://badgen.net/badge/Dependabot/enabled/green?icon=dependabot">
  </a>
  <a href="https://github.com/prettier/prettier">
    <img alt="code style: prettier" src="https://img.shields.io/badge/code_style-prettier-ff69b4.svg?style=flat-square">
  </a>
</p>

## Usage

```
...

plugins:
  - serverless-idempotency-helper

...
```

## Background

Notes on implementation:

- The atime and mtime of files contained in zip files affect the commit hash, but aren't relevant to our deployment logic. So, atime/mtime should be washed out of the zip, so the commit hash delta reflects our desired behavior.
- The serverless-bundle and serverless-webpack plugins generate new source code (the packed source code) on every deployment. As such, atime/mtime is always an issue when using these plugins.
- The inclusion of directory entries in function archives causes the sha256 checksum to be different. To skip adding directory entries, the "-D" flag is set for the zip command. This causes no functional change to the archive, but allows for an idempotent zip. This behavior was observed on ubuntu, and it was not observed on osx... so on mac, the zip was idempotent without the "-D" flag, but on linux the "-D" flag was necessary.
- This plugin hooks into package:createDeploymentArtifacts. The user defined functions (from the functions key in serverless.yml) are repacked at this step. We need to repack these functions no later than this lifecycle event, because it's seemingly just before Serverless decides the commit hashes of these archives. If we repack later than this step, there will be a mismatch between the commit hash according to Serverless and the true commit hash, causing a CloudFormation failure. Previously this plugin had been written to change atime/mtime of pre-zipped webpack output, or pre-zipped source code, but repacking after all that seemed a more consistent approach.
- This plugin hooks into package:compileEvents. Just after this step is when custom, non-user-defined archives appear. A good example of this is the custom resource generated by turning on api gateway logging. When you enable logging via the provider.logging.restApi attribute, a custom resource with a corresponding lambda archive is generated at this step. We need to repack that archive just like the user defined functions archives, or we will be non-idempotent. This requirement is what drove repacking all archives; instead of changing atime/mtime of pre-zipped files for functions and repacking custom zips, we repack everything; the consistent approach is seen as worthwhile.

## License

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

See [LICENSE](LICENSE) for full details.
