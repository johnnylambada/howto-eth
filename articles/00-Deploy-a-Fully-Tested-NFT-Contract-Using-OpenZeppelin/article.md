# Deploy a Fully Tested NFT Contract Using OpenZeppelin

- [Deploy a Fully Tested NFT Contract Using OpenZeppelin](#deploy-a-fully-tested-nft-contract-using-openzeppelin)
  - [Introduction](#introduction)
  - [What are we going to build?](#what-are-we-going-to-build)
  - [A note about my environment](#a-note-about-my-environment)
    - [shell.nix](#shellnix)
    - [devbin (Development Binaries)](#devbin-development-binaries)
    - [devlog](#devlog)
    - [.envrc](#envrc)
    - [.vscode](#vscode)
  - [Let's get started](#lets-get-started)
    - [Install npm and truffle](#install-npm-and-truffle)
    - [Initialize truffle](#initialize-truffle)
      - [`truffle-config.js`](#truffle-configjs)
      - [`contracts/Migrations.sol` (can be ignored)](#contractsmigrationssol-can-be-ignored)
      - [`migrations/1_initial_migration.js` (can be gignored)](#migrations1_initial_migrationjs-can-be-gignored)
      - [`test/.gitkeep` (can be gignored)](#testgitkeep-can-be-gignored)
    - [Use the latest version of the solidity compiler](#use-the-latest-version-of-the-solidity-compiler)
    - [Install the rest of the environment](#install-the-rest-of-the-environment)
      - [`@truffle/hdwallet-provider`](#trufflehdwallet-provider)
      - [`mocha` and `chai`](#mocha-and-chai)
      - [`ganache-cli`](#ganache-cli)
      - [`@openzeppelin` `contracts`, `test-environment` and `test-helpers`](#openzeppelin-contracts-test-environment-and-test-helpers)
- [MORE HERE](#more-here)
- [TODO](#todo)
    - [Congratulations #1](#congratulations-1)
  - [](#)

## Introduction

This article is the first in a series of articles that describe, in exacting detail, how I built a series of solidity contracts. I'm not going to go into a lot of theory, and I'm not going to try to sell you on how awesome ethereum, bitcoin, blockchain, etc are. There are plenty of other articles that do that. This series of articles will be a straight-forward description of what was built, how it works, and why I built it the way I did.

One thing to understand -- writing solidity contracts is very different from writing other software for two important reasons:

1. If you do your job well, there will be a lot of money flowing through your contract. This *WILL* attract nefarious individuals who would like to steal that money. You may be thinking "this is also true for banks". However, unlike banks your code is available to anyone who wants to review it in bytecode format. Bottom line -- you MUST test your code, and you MUST try to think of all the ways someone will subvert it to steal the contract's value.
2. Once a contract is deployed to the main Ethereum network it is immutable unless you specificaly build your contract so it can be upgradelable. [OpenZeppelin Upgrade Plugins](https://docs.openzeppelin.com/learn/upgrading-smart-contracts) can be used to mitigate this problem at the cost of gas and complexity. 

TLDR: Testting your contract is essential. Test Driven Development is the way to go here, and this series of articles will use that approach.

## What are we going to build?

In this article we'll concentrate on building a fully tested NFT contract using OpenZeppelin. It'll be quite easy because OpenZeppelin has already written all of the code and tests for the contract, so we'll simply use that code and those tests. We'll take some time to explore some of the OpenZeppelin code along the way.

## A note about my environment

You'll find several files in the top level directory of the github repositiory that are part of my development environment. I describe them here for completeness.

### shell.nix

I use nix to manage the packages on my development computer. This nix expression makes node version 12 available to the project.

```nix
{ pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    nativeBuildInputs = [ pkgs.nodejs-12_x ];
}
```

### devbin (Development Binaries)

There are a series of shell scripts in this directory that are added to my path when I `cd` into the directory. They're there to help me rember how to do specific commands that I need to do. There is also a `help` script that prints out a simple description of what each script in `devbin` does.

### devlog

Any log files that I may want review go here. For this project, this is mainly the `ganache-cli` log.

### .envrc

I use `direnv` to set up the specific environment necessary to work on this project when I `cd` into it.

```
use_nix  ①
PATH_add devbin ②
export NFTCAR_DIR=$PWD ③
```

1. `use-nix` is a `direnv` directive that tells `direnv` to invoke the `shell.nix` expression.
2. This adds the `devbin` path to my path.
3. This adds a new environment variable to my shell.

### .vscode

I use vscode as my IDE. The `.vscode` directory contains all of the configuration files for the IDE.

## Let's get started

As I mentioned in the previous section, I use `nix` to curate my development environment for this project. You are free to use anything you like. `npm -g` or `nvm` are both fine choices. I'll assume you have `node` version 12 available to you on the command line along with `npm` and `npx`.

### Install npm and truffle

```bash
npm init -y  # ①

cat >> .gitignore # ②
node_modules/
build/
^d ③
npm install --save-dev truffle # ④

```

1. This sets up node in this directory.
2. Update `.gitignore` to so that we don't push compile results or `node_modules` to our git repository.
3. Don't forget to hit [ctrl][d] to close the `cat` command before typing the next command.
4. Install the [Truffle Suite](https://www.trufflesuite.com/) locally.

### Initialize truffle

```bash
npx truffle init
```

This command initializes a new and empty ethereum project in the local directory. It creates a series of files that we'll examine below.

#### `truffle-config.js`

This is the most important file that was created by `npx truffle init`. It's heavily commented, but the uncommented version is shown below so you can get a sense of what you're dealing with.

```javascript
module.exports = {①
  networks: { ②
  },
  mocha: {③
  },
  compilers: {④
    solc: {
    }
  },
  db: {
    enabled: false ⑤
  }
};
```

1. This javascript essentially exports a single object containing the configuration for the local truffle. It's comprised of four separate sections: `networks`, `mocha`, `compilers`, and `db`.
2. Networks define how you connect to your ethereum client and let you set the defaults web3 uses to send transactions.
3. We use the `mocha` testing library for testing Ethereum contracts using truffle. You can set default `mocha` options here. A good one to set is the `timeout` option as shown in the example file (just uncomment it). Without it any breakpoints in your test code will fail your test due to a timeout.
4. The `compilers` section is where you configure your compiler. We'll get back to this in a minute.
5. Truffle starts off with the truffle database turned off. We're going to leav it that way, but you may want to turn it on. Check out [these](https://www.npmjs.com/package/@truffle/db) [articles](https://www.trufflesuite.com/blog/introducing-truffle-db-part-1) for more information.

#### `contracts/Migrations.sol` (can be ignored)

This is a solidity contract that truffle uses as part of the process of migrating (think installing) your contract to the Ethereum block chain. I've never modified this file.

#### `migrations/1_initial_migration.js` (can be gignored)

This is a javascript file that truffle also uses as part of the migration process. I've never modified this file either.

#### `test/.gitkeep` (can be gignored)

This file just exists so that the `test` directory, which would otherwise be empty, is stored in your git repository. It never changes and can be deleted once you have tests in the `test` directory.

### Use the latest version of the solidity compiler

Modify the `truffle-config.js` file as follows. If you're not used to reading the output of the `diff` command, this snippet is essentially saying to change the `0.5.1` to `0.8.6` in the `compilers.solc` object. [Solidity 0.8.6](https://docs.soliditylang.org/en/v0.8.6/) is the latest version as of this writing.

```diff
diff --git a/truffle-config.js b/truffle-config.js
index a707377..8e8d0b4 100644
--- a/truffle-config.js
+++ b/truffle-config.js
@@ -82,7 +82,7 @@ module.exports = {
   // Configure your compilers
   compilers: {
     solc: {
-      // version: "0.5.1",    // Fetch exact version from solc-bin (default: truffle's version)
+      version: "0.8.6",    // Fetch exact version from solc-bin (default: truffle's version)
       // docker: true,        // Use "0.5.1" you've installed locally with docker (default: false)
       // settings: {          // See the solidity docs for advice about optimization and evmVersion
       //  optimizer: {
```

### Install the rest of the environment

There are several more NPM modules we need to install. We install them below.

#### `@truffle/hdwallet-provider`
```bash
npm install --save-dev @truffle/hdwallet-provider
```

[Truffle's hdwallet-provider](https://www.npmjs.com/package/@truffle/hdwallet-provider) is a Web3 provider used to signb transactions for addresses derived from 12 or 24 word mnemonics.

#### `mocha` and `chai`
```bash
npm install --save-dev mocha chai
```

[Mocha](https://mochajs.org/) is a JavaScript test framework for Node. [Chai](https://www.chaijs.com/) is a BDD / TDD assertion library.


#### `ganache-cli`
```bash
npm install --save-dev ganache-cli
```

[ganache-cli](https://github.com/trufflesuite/ganache-cli) is part of the Truffle suite. It is the command line version of Ganache and provides a test blockchain on your local machine.


#### `@openzeppelin` `contracts`, `test-environment` and `test-helpers`
```bash
npm install --save-dev @openzeppelin/contracts @openzeppelin/test-environment @openzeppelin/test-helpers
```

* [contracts](https://docs.openzeppelin.com/contracts/4.x/)  A library of well tested smart contracts. The library includes the ERC721 contract that we'll use in this article.
* [test-environment](https://docs.openzeppelin.com/test-environment/0.1/)  Sets up a great test environment for testing your contracts.  Accounts, simple connections to Contracts, etc. It takes a lot of boilerplate out of your tests.
* [test-helpers](https://docs.openzeppelin.com/test-helpers/0.5/) is an assertion library for smart contract testing.


# MORE HERE

# TODO
* migrations/2_deploy.js
* Set up development network in truffle-config.js
* Update test environment to use `mocha`:
```diff
diff --git a/package.json b/package.json
index ac7b362..87dee76 100644
--- a/package.json
+++ b/package.json
@@ -4,7 +4,7 @@
   "description": "",
   "main": "index.js",
   "scripts": {
-    "test": "echo \"Error: no test specified\" && exit 1"
+    "test": "truffle compile && mocha --exit --recursive"
   },
   "keywords": [],
   "author": "",
```
* [Add rinkeby network to truffle.conf](https://github.com/johnnylambada/nftcar/commit/44a478bea13a5d8976cbfa4368d3ec264f7ee5fd)  / [see this commit too](https://github.com/johnnylambada/nftcar/commit/27a98183a9a4a1450975f2bfb6fa8041c86e69d1)

### Congratulations #1

If you've made it this far, you have a perfect starting point for any new solidity project you want! The only thing missing is an actual contract and some tests. That's what the next section is all about!

## 