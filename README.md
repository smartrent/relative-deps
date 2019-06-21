# relative-deps

**WORK IN PROGRESS!**

Installs local dependencies for optimal developer experience

---

# Why

### The problem

Working on libraries that have examples embedded in the GitHub repo is usually tricky, as the examples are usually against the public, published version of the library. The version that is mentioned in their `package.json`. Often not the latest, definitely not the version currently being checked out and worked out.

This complicates many things, like the turn around time of local development (either publish first, or locally re-build and install the dependency). And it complicates CI for similar reasons.

### Alternative solutions

There are a few existing solutions, but they have their own limitations:

- `yarn link` / `npm link`. These work only if there are no peer dependencies involved. If there are peer dependencies, the linked library looks it up in it's _own_ `node_modules`, instead of the `node_modules` of the hosting project, where it would normally be looked up. This results in peer dependencies ending up "twice" in the dependency tree, which results in unexpected results.
- `yarn workspaces`. Those solve the above issue by putting all dependencies in one large root level `node_modules`, but, the setup is quite obtrusive and it introduces new problems, for example if the examples use different versions of the same libraries, this blows up quickly.

### How is relative deps different?

Relative deps doesn't fight the problem but tries to emulate a "normal" install. It builds the "linked" library on `postinstall`, packs it, and unpacks it in the `node_modules` of the hosting project. Since there is no linking, or shared `node_modules` involved, the folder structure ends up to be exactly the same as if the thing was installed directly from `yarn` / `npm`. Which avoids a plethora of problems.

Since building a linked package every time `yarn install` is run is expensive, this tool will take a hash of the directory contents of the library first, and only build and install if something changed.

# Installation

Install `relative-deps` as developer dependency. Either in the hosting project, or somewhere higher in the directory structure:

`yarn add -D relative-deps`.

In the hosting project, add the following `package.json` script:

`"postinstall": "yarn relative-deps"`

This will re-install any relative dependency if needed when running `yarn install`.

Optionally, you can add this step also for more scripts, for example:

```json
{
  "name": "mobx-react-demo",
  "scripts": {
    "postinstall": "yarn relative-deps",
    "prestart": "yarn relative-deps",
    "prebuild": "yarn relative-deps",
    "pretest": "yarn relative-deps"
  }
}
```

# Adding a relative dependency

To use a relative dependency, add it's name and relative path under the `relativeDependencies` top-level section in the `package.json` of the hosting package. For example:

````json
{
  "name": "mobx-react-demo",
  "relativeDependencies": {
    "mobx-react": "../../"
  }
}

Optionally, you can install a relative dependency as normal dependency as well. The benefit of this is that anybody that checks out the project, but doesn't have a checkout of the targeted library, gets the normally published version. For example:

```json
{
  "name": "mobx-react-demo",
  "relativeDependencies": {
    "mobx-react": "../../"
  },
  "dependencies": {
    "mobx-react": "^4.0.0"
  }
}
````

# How

Roughly, it works like this:

```
- pre: yarn.lock exists or die
- read relativeDeps from nearest package.json
- doesn't exist? warn & exit
- for each relativeDep:
- check if target path exists
  - if not, do we have the module from normal install?
  - yes: warn
  - no: error
- if target path exists, does it have node modules?
  - no: run yarn / npm install (guess which one)
- find last modified timestamp of all files in target dir
  (excluding node_modules, .git, excluding the directory that contains the calling project if applicable, only use git versioned files)
- take hash and store / compare with stored
- if changed:
  - run yarn / npm build
  - run pack
  - extract package (mind scoped package names!)
  - run yarn install --no-dev-deps in target dir
- done
```

# Future features

PRs are welcome!

- [ ] support `relativedeps add <path>`
- [ ] support `npm` as well
- [ ] support scoped package names
- [ ] support `relativedeps --init` to setup postinstall / build hooks
- [ ] support `relativedeps --watch` to watch project and automatically install!
- [ ] support more types of relative paths, such as git urls
- [ ] factor out script to unpack an arbirtrary package.tgz