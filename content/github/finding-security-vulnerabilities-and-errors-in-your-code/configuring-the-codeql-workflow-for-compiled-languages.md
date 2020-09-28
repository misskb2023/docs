---
title: Configuring the CodeQL workflow for compiled languages
shortTitle: Configuring for compiled languages
intro: 'You can configure how {{ site.data.variables.product.prodname_dotcom }} uses the {{ site.data.variables.product.prodname_codeql_workflow }} to scan code written in compiled languages for vulnerabilities and errors.'
product: '{{ site.data.reusables.gated-features.code-scanning }}'
permissions: 'People with write permissions to a repository can configure {{ site.data.variables.product.prodname_code_scanning }} for the repository.'
redirect_from:
  - /github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-code-scanning-for-compiled-languages
  - /github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-the-codeql-action-for-compiled-languages
versions:
  free-pro-team: '*'
  enterprise-server: '>=2.22'
---

{{ site.data.reusables.code-scanning.beta }}
{{ site.data.reusables.code-scanning.enterprise-enable-code-scanning-actions }}

### About the {{ site.data.variables.product.prodname_codeql_workflow }} and compiled languages

You enable {{ site.data.variables.product.prodname_dotcom }} to run {{ site.data.variables.product.prodname_code_scanning }} for your repository by adding a {{ site.data.variables.product.prodname_actions }} workflow to the repository. For {{ site.data.variables.product.prodname_codeql }} {{ site.data.variables.product.prodname_code_scanning }}, you add the {{ site.data.variables.product.prodname_codeql_workflow }}. For more information, see "[Enabling {{ site.data.variables.product.prodname_code_scanning }} for a repository](/github/finding-security-vulnerabilities-and-errors-in-your-code/enabling-code-scanning-for-a-repository)."

{{ site.data.reusables.code-scanning.edit-workflow }} 
For general information about configuring {{ site.data.variables.product.prodname_code_scanning }} and editing workflow files, see "[Configuring {{ site.data.variables.product.prodname_code_scanning }}](/github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-code-scanning)" and  "[Learn {{ site.data.variables.product.prodname_actions }}](/actions/learn-github-actions)."

###  About autobuild for {{ site.data.variables.product.prodname_codeql }}

Code scanning works by running queries against one or more databases. Each database contains a representation of all of the code in a single language in your repository. For the compiled languages C/C++, C#, and Java, the process of populating this database involves building the code and extracting data. In contrast to the other compiled languages, CodeQL can generate a database for Go without building the code.

{{ site.data.reusables.code-scanning.autobuild-compiled-languages }}

If your workflow uses a `language` matrix, `autobuild` attempts to build each of the compiled languages listed in the matrix. Without a matrix `autobuild` attempts to build the supported compiled language that has the most source files in the repository. With the exception of Go, analysis of other compiled languages in your repository will fail unless you supply explicit build commands.

{% note %}

**Note**: If you use self-hosted runners for {{ site.data.variables.product.prodname_actions }}, you may need to install additional software to use the `autobuild` process. Additionally, if your repository requires a specific version of a build tool, you may need to install it manually. For more information, see "[Specifications for {{ site.data.variables.product.prodname_dotcom }}-hosted runners](/actions/reference/specifications-for-github-hosted-runners/#supported-software)".

{% endnote %}

#### C/C++

| Supported system type | System name |
|----|----|
|  Operating system | Windows and Linux |
| Build system | Autoconf, CMake, qmake, Meson, Waf, SCons, and Linux Kbuild |

The behavior of the `autobuild` step varies according to the operating system that the extraction runs on. On Windows, the step has no default actions. On Linux, this step reviews the files present in the repository to determine the build system used:

1. Look for a build system in the root directory.
2. If none are found, search subdirectories for a unique directory with a build system for C/C++.
3. Run an appropriate command to configure the system. 

#### C#

| Supported system type | System name |
|----|----|
|  Operating system | Windows and Linux |
| Build system | .NET and MSbuild, as well as build scripts |

The `autobuild` process attempts to autodetect a suitable build method for C# using the following approach:

1. Invoke `dotnet build` on the solution (`.sln`) or project (`.csproj`) file closest to the root.
2. Invoke `MSbuild` (Linux) or `MSBuild.exe` (Windows) on the solution or project file closest to the root.
If `autobuild` detects multiple solution or project files at the same (shortest) depth from the top level directory, it will attempt to build all of them.
3. Invoke a script that looks like a build script—_build_ and _build.sh_ (in that order, for Linux) or _build.bat_, _build.cmd_, _and build.exe_ (in that order, for Windows).

#### Java

| Supported system type | System name |
|----|----|
|  Operating system | Windows, macOS and Linux (no restriction) |
| Build system | Gradle, Maven and Ant |

The `autobuild` process tries to determine the build system for Java codebases by applying this strategy:

1. Search for a build file in the root directory. Check for Gradle then Maven then Ant build files.
2. Run the first build file found. If both Gradle and Maven files are present, the Gradle file is used.
3. Otherwise, search for build files in direct subdirectories of the root directory. If only one subdirectory contains build files, run the first file identified in that subdirectory (using the same preference as for 1). If more than one subdirectory contains build files, report an error.

### Adding build steps for a compiled language

{{ site.data.reusables.code-scanning.autobuild-add-build-steps }} For information on how to edit the workflow file, see  "[Configuring {{ site.data.variables.product.prodname_code_scanning }}](/github/finding-security-vulnerabilities-and-errors-in-your-code/configuring-code-scanning#editing-a-code-scanning-workflow)."

After removing the `autobuild` step, uncomment the `run` step and add build commands that are suitable for your repository. The workflow `run` step runs command-line programs using the operating system's shell. You can modify these commands and add more commands to customize the build process. 

``` yaml
- run: |
  make bootstrap
  make release
```

For more information about the `run` keyword, see "[Workflow syntax for {{ site.data.variables.product.prodname_actions }}](/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsrun)."

If your repository contains multiple compiled languages, you can specify language-specific build commands. For example, if your repository contains C/C++, C# and Java, and `autobuild` correctly builds C/C++ and C# but fails to build Java, you could use the following configuration in your workflow, after the `init` step. This specifies build steps for Java while still using `autobuild` for C/C++ and C#:

```yaml
- if: matrix.language == 'cpp' || matrix.language == 'csharp' 
  name: Autobuild
  uses: github/codeql-action/autobuild@v1

- if: matrix.language == 'java' 
  name: Build Java
  run: |
    make bootstrap
    make release
```

For more information about the `if` conditional, see "[Workflow syntax for GitHub Actions](/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsif)."

For more tips and tricks about why `autobuild` won't build your code, see "[Troubleshooting the {{ site.data.variables.product.prodname_codeql }} workflow](/github/finding-security-vulnerabilities-and-errors-in-your-code/troubleshooting-the-codeql-workflow)."

If you added manual build steps for compiled languages and {{ site.data.variables.product.prodname_code_scanning }} is still not working on your repository, contact {{ site.data.variables.contact.contact_support }}.