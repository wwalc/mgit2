# mgit2

<a href="https://www.npmjs.com/package/mgit2"><img src="https://img.shields.io/npm/v/mgit2.svg" alt="mgit2 npm package badge"></a>
<a href="https://travis-ci.org/cksource/mgit2"><img src="https://img.shields.io/travis/cksource/mgit2/master.svg" alt="build status badge"></a>
<a href="https://codeclimate.com/github/cksource/mgit2/coverage"><img src="https://img.shields.io/codeclimate/coverage/github/cksource/mgit2.svg" alt="mgit2 coverage badge"></a>
<a href="https://david-dm.org/cksource/mgit2"><img src="https://img.shields.io/david/cksource/mgit2.svg" alt="mgit2 dependencies status badge"></a>
<a href="https://david-dm.org/cksource/mgit2?type=dev"><img src="https://img.shields.io/david/dev/cksource/mgit2.svg" alt="mgit2 devDependencies status badge"></a>

Multi-repo manager for git. A tool for managing projects build using multiple repositories.

mgit2 is designed to work with [Lerna](https://github.com/lerna/lerna) (*"A tool for managing JavaScript projects with multiple packages."*) out of the box, hence, it mixes the "package" and "repository" concepts. In other words, every repository is meant to be a single [npm](https://npmjs.com) package. It doesn't mean that you must use it with Lerna and npm, but don't be surprised that mgit2 talks about "packages" and works best with npm packages.

## Installation

```bash
sudo npm install -g mgit2
```

Use:

```bash
mgit --help
```

**Note:** The command is called `mgit`.

## Usage

First, create a configuration file `mgit.json`:

```json
{
  "dependencies": {
    "@ckeditor/ckeditor5-engine": "ckeditor/ckeditor5-engine",
    "mgit": "cksource/mgit"
  }
}
```

And run `mgit bootstrap` to clone all the repositories. By default, they will be cloned to `<cwd>/packages/` directory:

```
packages/
  ckeditor5-engine/
  mgit/
```

## Configuration

CLI options:

```
--recursive                 Whether to install dependencies recursively.
                            Needs to be used together with --repository-include. Only packages
                            matching these patterns will be cloned recursively.

                            Default: false.

--packages                  Directory to which all repositories will be cloned.

                            Default: '<cwd>/packages/'

--resolver-path             Path to a custom repository resolver function.

                            Default: '@mgit2/lib/default-resolver.js'.

--resolver-url-template     Template used to generate repository URL out of a
                            simplified 'organization/repository' format of the dependencies option.

                            Default: 'git@github.com:${ path }.git'.

--resolver-directory-name   Defines how the target directory (where the repository will be cloned)
                            is resolved. Supported options are: 'git' (default), 'npm'.

                            * If 'git' was specified, then the directory name will be extracted from
                            the git URL (e.g. for 'git@github.com:a/b.git' it will be 'b').
                            * If 'npm' was specified, then the package name will be used as a directory name.

                            This option can be useful when scoped npm packages are used and one wants to decide
                            whether the repository will be cloned to packages/@scope/pkgname' or 'packages/pkgname'.

                            Default: 'git'

--resolver-default-branch   The branch name to use if not specified in mgit.json dependencies.

                            Default: 'master'
```

All these options can also be specified in `mgit.json` (options passed through CLI takes precedence):

```js
{
	"packages": "/workspace/modules",
	"resolverDirectoryName": "npm",
	"resolverDefaultBranch": "dev",
	"dependencies": {
		"foo": "bar"
	}
}
```

### The `dependencies` option

This option specifies repositories which mgit is supposed to clone. It can also clone its dependencies recursively (see [Recursive cloning](#recursive-cloning)).

The dependency keys can be any strings, but it's recommended to use package names (e.g. npm package names, just like in `package.json`). The values are repository URLs which mgit will clone.

Examples:

```js
// Clone 'git@github.com:cksource/foo.git' and check out to 'master'.
{
	"foo": "git@github.com:cksource/foo.git"
}
```

```js
// Short format. Clone 'git@github.com:cksource/foo.git' and check out to branch 'dev'.
{
	"@cksource/foo": "cksource/foo#dev"
}
```

```js
// Clone 'https://github.com/cksource/foo.git' (via HTTPS) and check out to branch tag 'v1.2.3'.
{
	"foo": "https://github.com/cksource/foo.git#v1.2.3"
}
```

### Recursive cloning

When the `--recursive` option is used mgit will clone repositories recursively. First, it will clone the `dependencies` specified in `mgit.json` and, then, their `dependencies` and `devDependencies` specified in `package.json` files located in cloned repositories.

However, mgit needs to know repository URLs of those dependencies, as well as which dependencies to clone (usually, only the ones maintained by you). In order to configure that you need to use a custom repository resolver (`--resolver-path`).

Resolver is a simple Node.js module which exports the resolver function.

For example, assuming that you want to clone all `@ckeditor/ckeditor5-*` packages, your resolver could look like this:

```js
'use strict';

const parseRepositoryUrl = require( 'mgit2/lib/utils/parserepositoryurl' );

/**
 * Resolves repository URL for a given package name.
 *
 * @param {String} packageName Package name.
 * @param {Options} data.options The options object.
 * @returns {Repository|null}
 */
module.exports = function resolver( packageName, options ) {
	// If package name starts with '@ckeditor/ckeditor5-*' clone it from 'ckeditor/ckeditor5-*'.
	if ( packageName.startsWith( '@ckeditor/ckeditor5-' ) ) {
		repositoryUrl = packageName.slice( 1 );

		return parseRepositoryUrl( repositoryUrl );
	}

	// Don't clone any other dependencies.
	return null;
};
```

You can also check the [default resolver](https://github.com/cksource/mgit2/blob/master/lib/default-resolver.js) used by mgit and [the config object definition](https://github.com/cksource/mgit2/blob/master/lib/utils/getconfig.js).

### Cloning repositories on CI servers

CI servers, such as Travis, can't clone repositories using Git URLs (such as `git@github.com:cksource/mgit.git`). By default, mgit uses Git URLs because it assumes that you'll want to commit to these repositories (and don't want to be asked for a password every time).

If you need to run mgit on a CI server, then configure it to use HTTP URLs:

```bash
mgit --resolver-url-template="https://github.com/\${ path }.git"
```

You can also use full HTTPs URLs to configure `dependencies` in your `mgit.json`.

## Commands

```
mgit [command]
```

### bootstrap

Installs missing packages (i.e. clone them) and check them out to correct branches.

This command will not change existing repositories, so you can always safely use it. It's useful to bootstrap the project initially, but later you'll rather want to use `mgit update`.

Example:

```bash
mgit bootstrap --recursive --resolver=path ./dev/custom-repository-resolver.js
```

### update

Updates dependencies. Switches repositories to correct branches (specified in `mgit.json`) and pulls changes.

If any dependency is missing, the command will install it too.

This command does not touch repositories in which there are uncommitted changes.

Examples:

```bash
mgit update --recursive
```

### exec

For every cloned repository executes the specified shell command.

Example:

```bash
mgit exec 'git status'

# Executes `git status` command on each repository.
```

During the task execution, `cwd` is set to the repository path:

```bash
mgit exec 'echo `pwd`'

# /home/mgit/packages/organization/repository-1
# /home/mgit/packages/organization/repository-2
```


### save-hashes

Saves hashes of packages in `mgit.json`. It allows to easily fix project to a specific state.

Example:

```bash
mgit save-hashes
```

## Projects using mgit2

* [CKEditor 5](https://github.com/ckeditor/ckeditor5)
