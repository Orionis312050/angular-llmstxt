---
title: "CLI command reference"
---

### CLI command reference
Source: https://angular.dev/cli

#### `ng `
Command: ``



#### `ng add`
Command: `ng add <collection>`

Adds support for an external library to your project.

Adds the npm package for a published library to your workspace, and configures
the project in the current working directory to use that library, as specified by the library's schematic.
For example, adding `@angular/pwa` configures your project for PWA support:

```bash
ng add @angular/pwa
```


#### `ng analytics`
Command: `ng analytics`

Configures the gathering of Angular CLI usage metrics.

You can help the Angular Team to prioritize features and improvements by permitting the Angular team to send command-line command usage statistics to Google.
The Angular Team does not collect usage statistics unless you explicitly opt in. When installing the Angular CLI you are prompted to allow global collection of usage statistics.
If you say no or skip the prompt, no data is collected.

### What is collected?

Usage analytics include the commands and selected flags for each execution.
Usage analytics may include the following information:

- Your operating system \(macOS, Linux distribution, Windows\) and its version.
- Package manager name and version \(local version only\).
- Node.js version \(local version only\).
- Angular CLI version \(local version only\).
- Command name that was run.
- Workspace information, the number of application and library projects.
- For schematics commands \(add, generate and new\), the schematic collection and name and a list of selected flags.
- For build commands \(build, serve\), the builder name, the number and size of bundles \(initial and lazy\), compilation units, the time it took to build and rebuild, and basic Angular-specific API usage.

Only Angular owned and developed schematics and builders are reported.
Third-party schematics and builders do not send data to the Angular Team.


#### `ng build`
Command: `ng build [project]`

Compiles an Angular application or library into an output directory named dist/ at the given output path.

The command can be used to build a project of type "application" or "library".
Depending on the project type, the CLI invokes either the application builder (`@angular/build:application`) or the library builder (`@angular/build:library`), each supporting its own set of options.

The application builder uses the [esbuild](https://esbuild.github.io/) build tool, with default configuration options specified in the workspace configuration file (`angular.json`) or with a named alternative configuration.
A "development" configuration is created by default when you use the CLI to create the project, and you can use that configuration by specifying the `--configuration development`.

The configuration options generally correspond to the command options.
You can override individual configuration defaults by specifying the corresponding options on the command line.
The command can accept option names given in dash-case.
Note that in the configuration file, you must specify names in camelCase.

Some additional options can only be set through the configuration file,
either by direct editing or with the `ng config` command.
These include `assets`, `styles`, and `scripts` objects that provide runtime-global resources to include in the project.
Resources in CSS, such as images and fonts, are automatically written to the `media` directory of the browser output
(configurable with the `outputPath.media` option) and fingerprinted when `outputHashing` is set to `all` or `media`.

For further details, see [Workspace Configuration](reference/configs/workspace-config).


#### `ng cache`
Command: `ng cache`

Configure persistent disk cache and retrieve cache statistics.

Angular CLI saves a number of cachable operations on disk by default.

When you re-run the same build, the build system restores the state of the previous build and re-uses previously performed operations, which decreases the time taken to build and test your applications and libraries.

To amend the default cache settings, add the `cli.cache` object to your [Workspace Configuration](reference/configs/workspace-config).
The object goes under `cli.cache` at the top level of the file, outside the `projects` sections.

```jsonc
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "cli": {
    "cache": {
      // ...
    },
  },
  "projects": {},
}
```

For more information, see [cache options](reference/configs/workspace-config#cache-options).

### Cache environments

By default, disk cache is only enabled for local environments. The value of environment can be one of the following:

- `all` - allows disk cache on all machines.
- `local` - allows disk cache only on development machines.
- `ci` - allows disk cache only on continuous integration (CI) systems.

To change the environment setting to `all`, run the following command:

```bash
ng config cli.cache.environment all
```

For more information, see `environment` in [cache options](reference/configs/workspace-config#cache-options).

<div class="alert is-helpful">

The Angular CLI checks for the presence and value of the `CI` environment variable to determine in which environment it is running.

</div>

### Cache path

By default, `.angular/cache` is used as a base directory to store cache results.

To change this path to `.cache/ng`, run the following command:

```bash
ng config cli.cache.path ".cache/ng"
```


#### `ng completion`
Command: `ng completion`

Set up Angular CLI autocompletion for your terminal.

Setting up autocompletion configures your terminal, so pressing the `<TAB>` key while in the middle
of typing will display various commands and options available to you. This makes it very easy to
discover and use CLI commands without lots of memorization.

![A demo of Angular CLI autocompletion in a terminal. The user types several partial `ng` commands,
using autocompletion to finish several arguments and list contextual options.
](assets/images/guide/cli/completion.gif)

## Automated setup

The CLI should prompt and ask to set up autocompletion for you the first time you use it (v14+).
Simply answer "Yes" and the CLI will take care of the rest.

```
$ ng serve
? Would you like to enable autocompletion? This will set up your terminal so pressing TAB while typing Angular CLI commands will show possible options and autocomplete arguments. (Enabling autocompletion will modify configuration files in your home directory.) Yes
Appended `source <(ng completion script)` to `/home/my-username/.bashrc`. Restart your terminal or run:

source <(ng completion script)

to autocomplete `ng` commands.

# Serve output...
```

If you already refused the prompt, it won't ask again. But you can run `ng completion` to
do the same thing automatically.

This modifies your terminal environment to load Angular CLI autocompletion, but can't update your
current terminal session. Either restart it or run `source <(ng completion script)` directly to
enable autocompletion in your current session.

Test it out by typing `ng ser<TAB>` and it should autocomplete to `ng serve`. Ambiguous arguments
will show all possible options and their documentation, such as `ng generate <TAB>`.

## Manual setup

Some users may have highly customized terminal setups, possibly with configuration files checked
into source control with an opinionated structure. `ng completion` only ever appends Angular's setup
to an existing configuration file for your current shell, or creates one if none exists. If you want
more control over exactly where this configuration lives, you can manually set it up by having your
shell run at startup:

```bash
if [[ $- == *i* ]]; then
  source <(ng completion script)
fi
```

This is equivalent to what `ng completion` will automatically set up, and gives power users more
flexibility in their environments when desired.

## Platform support

Angular CLI supports autocompletion for the Bash and Zsh shells on MacOS and Linux operating
systems. On Windows, Git Bash and [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/)
using Bash or Zsh are supported.

## Global install

Autocompletion works by configuring your terminal to invoke the Angular CLI on startup to load the
setup script. This means the terminal must be able to find and execute the Angular CLI, typically
through a global install that places the binary on the user's `$PATH`. If you get
`command not found: ng`, make sure the CLI is installed globally which you can do with the `-g`
flag:

```bash
npm install -g @angular/cli
```


#### `ng config`
Command: `ng config [json-path] [value]`

Retrieves or sets Angular configuration values in the angular.json file for the workspace.

A workspace has a single CLI configuration file, `angular.json`, at the top level.
The `projects` object contains a configuration object for each project in the workspace.

You can edit the configuration directly in a code editor,
or indirectly on the command line using this command.

The configurable property names match command option names,
except that in the configuration file, all names must use camelCase,
while on the command line options can be given dash-case.

For further details, see [Workspace Configuration](reference/configs/workspace-config).

For configuration of CLI usage analytics, see [ng analytics](cli/analytics).


#### `ng deploy`
Command: `ng deploy [project]`

Invokes the deploy builder for a specified project, the only project with a deploy target, or the project in the current working directory.

The command takes an optional project name, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, executes the `deploy` builder for the only project that has a `deploy` target,
or for the project in the current working directory.

To use the `ng deploy` command, use `ng add` to add a package that implements deployment capabilities to your favorite platform.
Adding the package automatically updates your workspace configuration, adding a deployment
[CLI builder](tools/cli/cli-builder).
For example:

```json
"projects": {
  "my-project": {
    ...
    "architect": {
      ...
      "deploy": {
        "builder": "@angular/fire:deploy",
        "options": {}
      }
    }
  }
}
```


#### `ng e2e`
Command: `ng e2e [project]`

Builds and serves an Angular application, then runs end-to-end tests.

#### `ng extract-i18n`
Command: `ng extract-i18n [project]`

Extracts i18n messages from source code.

#### `ng generate`
Command: `ng generate <schematic>`

Generates and/or modifies files based on a schematic.

#### `ng lint`
Command: `ng lint [project]`

Runs linting tools on Angular application code in a given project folder.

The command takes an optional project name, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, executes the `lint` builder for all projects.

To use the `ng lint` command, use `ng add` to add a package that implements linting capabilities. Adding the package automatically updates your workspace configuration, adding a lint [CLI builder](tools/cli/cli-builder).
For example:

```json
"projects": {
  "my-project": {
    ...
    "architect": {
      ...
      "lint": {
        "builder": "@angular-eslint/builder:lint",
        "options": {}
      }
    }
  }
}
```


#### `ng new`
Command: `ng new [name]`

Creates a new Angular workspace.

Creates and initializes a new Angular workspace with an initial application.

Provides interactive prompts for optional configuration, such as adding routing support.
All prompts can safely be allowed to default.

- The new workspace folder is given the specified project name, and contains configuration files at the top level.

- By default, the files for a new initial application (with the same name as the workspace) are placed in the `src/` subfolder.
- The new application's configuration appears in the `projects` section of the `angular.json` workspace configuration file, under its project name.

- Subsequent applications that you generate in the workspace reside in the `projects/` subfolder.

If you plan to have multiple applications in the workspace, you can create an empty workspace by using the `--no-create-application` option.
You can then use `ng generate application` to create an initial application.
This allows a workspace name different from the initial app name, and ensures that all applications reside in the `/projects` subfolder, matching the structure of the configuration file.


#### `ng run`
Command: `ng run <target>`

Runs an Architect target with an optional custom builder configuration defined in your project.

Architect is the tool that the CLI uses to perform complex tasks such as compilation, according to provided configurations.
The CLI commands run Architect targets such as `build`, `serve`, `test`, and `lint`.
Each named target has a default configuration, specified by an `options` object,
and an optional set of named alternate configurations in the `configurations` object.

For example, the `serve` target for a newly generated app has a predefined
alternate configuration named `production`.

You can define new targets and their configuration options in the `architect` section
of the `angular.json` file, which you can run from the command line using the `ng run` command.


#### `ng serve`
Command: `ng serve [project]`

Builds and serves your application, rebuilding on file changes.

#### `ng test`
Command: `ng test [project]`

Runs unit tests in a project.

Takes the name of the project, as specified in the `projects` section of the `angular.json` workspace configuration file.
When a project name is not supplied, it will execute for all projects.


#### `ng update`
Command: `ng update [packages..]`

Updates your workspace and its dependencies. See https://update.angular.dev/.

Perform a basic update to the current stable release of the core framework and CLI by running the following command.

```
ng update @angular/cli @angular/core
```

To update to the next beta or pre-release version, use the `--next` option.

To update from one major version to another, use the format

```
ng update @angular/cli@^<major_version> @angular/core@^<major_version>
```

We recommend that you always update to the latest patch version, as it contains fixes we released since the initial major release.
For example, use the following command to take the latest 21.x.x version and use that to update.

```
ng update @angular/cli@^21 @angular/core@^21
```

For detailed information and guidance on updating your application, see the interactive [Angular Update Guide](/update-guide).


#### `ng version`
Command: `ng version`

Outputs Angular CLI version.
