# Baulk - Minimal Package Manager for Windows

[![Master Branch Status](https://github.com/baulk/baulk/workflows/CI/badge.svg)](https://github.com/baulk/baulk/actions)

[简体中文](./ReadMe.zh-CN.md)

The design concept of Baulk comes from the [`devi`](https://github.com/fstudio/clangbuilder/blob/master/bin/devi.ps1) tool of [clangbuilder](https://github.com/fstudio/clangbuilder). devi is a simple package management tool developed based on PowerShell. It was originally used to solve the problem of software upgrade when building the LLVM/Clang build environment. Later, the Ports mechanism was implemented, which became A small package management tool. devi supports installation and uninstallation of upgrade packages, and also supports the creation of symbolic links to specific links directories. Such a strategy can effectively reduce the entries of the PATH environment variable, and with mklauncher can support commands that cannot use symbolic links, create its startup To links. The concept of devi is to avoid installing software that requires privileges and modify the operating system registry. And to avoid modifying operating system environment variables during software installation. Generally speaking, adding specific software to environment variables can indeed simplify the use, but as the number of installed software increases, environment variables will be easily overwritten, conflicts may occur when multiple versions of software are installed at the same time, and so on. From the birth of devi in ​​2018 to the present, I have a deep understanding of software management and reflection on devi, and devi has many problems that need to be solved, such as the download of the package does not support hash check, mklauncher needs to be run separately, Slower startup, etc. After thinking for a long time, I decided to implement a new package manager based on C++17 and [bela](https://github.com/fcharlie/bela), which is Baulk. Baulk integrates my technical accumulation over the past few years. I think that baulk is much better than devi. Due to the slower startup and execution of PowerShell, devi's execution speed is completely inferior to baulk. In fact, I am using Golang to re-implement When building the packaging tool [Bali](https://github.com/baulkbuild/bali) based on the Golang project written in PowerShell, I have the same feeling.

Windows is constantly improving. I also submitted a PR to the Windows Terminal. I hope that Baulk focuses on running on the new Windows. Therefore, when implementing Baulk, the error messages are all ANSI escaped (Bela is actually in the old The console supports ANSI Escape to Console API), and Baulk also adds the baulkterminal command to highly integrate with Windows Terminal. In addition, a script is added to support the user to modify the right-click menu and open the Windows Terminal that initializes the Baulk environment according to a specific startup path in the Explorer directory

## Usage

The command line parameters of baulk are roughly divided into three parts. The first part is `option`, which is used to specify or set some variables; the second part is `command`, which is the baulk subcommand, including installation and uninstallation, upgrade, update, freeze, unfreeze, etc. Command; the third part is the package name following the command. Of course, specific orders and specific analyses cannot be rigidly understood.

```txt
baulk - Minimal Package Manager for Windows
Usage: baulk [option] command pkg ...
  -h|--help        Show usage text and quit
  -v|--version     Show version number and quit
  -V|--verbose     Make the operation more talkative
  -F|--force       Turn on force mode. such as force update frozen package
  -P|--profile     Set profile path. default: Path\to\baulk\config\baulk.json
  -A|--user-agent  Send User-Agent <name> to server
  --https-proxy    Use this proxy. Equivalent to setting the environment variable 'HTTPS_PROXY'


Command:
  list             List all installed packages
  search           Search for available packages, or specific package details
  install          Install specific packages. upgrade if already installed.
  uninstall        Uninstall specific packages
  update           Update ports metadata
  upgrade          Upgrade all upgradeable packages
  freeze           Freeze specific package
  unfreeze         UnFreeze specific package
  b3sum            Calculate the BLAKE3 checksum of a file
  sha256sum        Calculate the SHA256 checksum of a file

```

|Command|Description|Remarks|
|---|---|---|
|list|View installed packages|N/A|
|search|Search for packages available in Bucket|baulk search command supports file name matching mode, for example `baulk search *` will search all packages in Bucket|
|install|Install specific packages|install also has other features, when the package has been installed will rebuild the launcher, when there is a new version of the package will upgrade it, `--force` will upgrade the frozen package|
|uninstall|Uninstall a specific package|N/A|
|update|update bucket metadata|similar to Ubuntu apt update command|
|upgrade|Update packages with new versions|`--force` can upgrade frozen packages|
|freeze|Freeze specific packages|Frozen packages cannot be upgraded in regular mode|
|unfreeze|Unfreeze the package|N/A|
|b3sum|Calculate the BLAKE3 hash of the file|N/A|
|sha256sum|Calculate the SHA256 hash of the file|N/A|

### Baulk configuration file

The default path of the baulk configuration file is `$ExecutableDir/../config/baulk.json`, which can be specified by setting the parameter `--profile`.

### Bucket management

In the bucket configuration file, we need to set `bucket`, which is used to store the source data of the baulk installation software. Buckets currently only support storage on git code hosting platforms, such as Github. To install software using baulk, there must be at least one `bucket`. The default bucket of baulk is [https://github.com/baulk/bucket](https://github.com/baulk/bucket). The bucket configuration is as follows:

**baulk.json**:

```json
{
    "bucket": [
        {
            "description": "Baulk default bucket",
            "name": "Baulk",
            "url": "https://github.com/baulk/bucket",
            "weights": 100
        }
    ]
}
```

In `bucket`, we designed the `weights` mechanism. In different `buckets`, if there is a package with the same name and the package version is the same, we will compare the `weights` of `bucket` The larger ones will be installed.

There is no command line support for adding buckets, just edit `baulk.json` and add according to its format.

To synchronize buckets, you can run the `baulk update` command. This is similar to `apt update`. The baulk synchronization bucket adopts the RSS synchronization mechanism, which is to obtain the latest commit information by requesting the bucket repository, compare the latest commitId with the last commitId recorded locally, and download the git archive to decompress it locally if they are inconsistent. The advantage of this mechanism is that it can support synchronization without installing git.

### Package management

The commands for the baulk management package include `install`, `uninstall`, `upgrade`, `freeze` and `unfreeze`, and `list` and `search`. Installing software using baulk is very simple, the command is as follows:

```shell
# Install cmake git and 7z
baulk install cmake git 7z
```

`baulk install` will install specific packages. During the installation process, baulk will read the metadata of the specific packages from the bucket. The format of these metadata is generally as follows:

```json
{
    "description": "CMake is an open-source, cross-platform family of tools designed to build, test and package software",
    "version": "3.17.2",
    "url": [
        "https://github.com/Kitware/CMake/releases/download/v3.17.2/cmake-3.17.2-win32-x86.zip",
        "https://cmake.org/files/v3.17/cmake-3.17.2-win32-x86.zip"
    ],
    "url.hash": "SHA256:66a68a1032ad1853bcff01778ae190cd461d174d6a689e1c646e3e9886f01e0a",
    "url64": [
        "https://github.com/Kitware/CMake/releases/download/v3.17.2/cmake-3.17.2-win64-x64.zip",
        "https://cmake.org/files/v3.17/cmake-3.17.2-win64-x64.zip"
    ],
    "url64.hash": "SHA256:cf82b1eb20b6fbe583487656fcd496490ffccdfbcbba0f26e19f1c9c63b0b041",
    "extension": "zip",
    "links": [
        "bin\\cmake.exe",
        "bin\\cmake-gui.exe",
        "bin\\cmcldeps.exe",
        "bin\\cpack.exe",
        "bin\\ctest.exe"
    ]
}
```

baulk downloads the compressed package according to the URL set in the list. If there is a compressed package with the same name and the hash value matches locally, the local cache is used. baulk uses WinHTTP to download the compressed package, which currently supports HTTP Proxy. baulk allows no hashes to be set in the list. The hash of baulk is set to `HashPrefix:HashContent` format. If there is no hash prefix, the default is `SHA256`. The hash prefixes supported by Baulk are `SHA256` and `BLAKE3`.

In baulk, `extension` supports `zip`, `msi`, `7z`, `exe`, `tar`, and baulk executes the corresponding decompression program according to the type of `extension`. The extended decompression procedure is as follows:

|Extended|Unzip Program|Limited|
|---|---|---|
|`exe`|-|-|
|`zip`|Built-in, based on minizip|Does not support encryption, does not support Deflate64, Deflate64 can use `7z`|
|`msi`|Built-in, based on MSI API|-|
|`7z`|Priority:</br>baulk7z-Baulk distribution</br>7z-installed using baulk install</br>7z-environment variables in the format of `tar.*` cannot be decompressed once Completed, so it is recommended to use `tar` to decompress `tar.*` compressed package|
|`tar`|Priority:</br>baulktar-modern reconstruction of BaulkTar bsdtar</br>bsdtar-Baulk build</br>MSYS2 tar-carried by Git for Windows</br>Windows tar|Windows built-in Tar does not support xz (based on libarchive bsdtar), but bsdtar built by baulk does not support Deflate64 when decompressing zip|

In the manifest file, there may also be `links/launchers`, and baulk will create symbolic links for specific files according to the settings of `links`. With Visual Studio installed, baulk will create a launcher based on the `launchers` setting, if If Visual Studio is not installed, it will use `baulk-lnk` to create an analog launcher. If baulk runs on Windows x64 or ARM64 architecture, there will be some small differences, that is, the platform-related URL/Launchers/Links is preferred, as follows:

|Architecture|URL|Launchers|Links|Remarks|
|---|---|---|---|---|
|x86|url|launchers|links|-|
|x64|url64, url|launchers64, launchers|links64, links|If the launchers/links of different architectures have the same goal, you don’t need to set them separately|
|ARM64|urlamr64, url|launchersarm64, launchers|linksarm64, links|If launchers/links of different architectures have the same goal, you don’t need to set them separately|


Tips: In Windows, after starting the process, we can use `GetModuleFileNameW` to get the binary file path of the process, but when the process starts from the symbolic link, the path of the symbolic link will be used. If we only use `links` in baulk to create symbolic links to the `links` directory, there may be a problem that a specific `dll` cannot be loaded, so here we use the `launcher` mechanism to solve this problem.

When running `baulk install`, if the software has already been installed, there may be two situations. The first is that the software has not been updated, then `baulk` will rebuild `links` and `launchers`, which is applicable to different packages. In the same `links`/`launchers` installation, overwriting needs to be restored. If there is an update to the software, baulk will install the latest version of the specified package.

`baulk uninstall` will remove the package and the created launcher, symbolic link. `baulk upgrade` By searching for already installed packages, if a new version of the corresponding package exists, install the new version.

There is also a `freeze/unfreeze` command in baulk. `baulk freeze` will freeze specific packages. Using `baulk install/baulk upgrade` will skip the upgrade of these packages. However, if `baulk install/baulk upgrade` is used The `--force/-f` parameter will force the upgrade of the corresponding package. We can also use `baulk unfreeze` to unfreeze specific packages.

In baulk, we can use `baulk search pattern` to search for packages added in the bucket, where `pattern` is based on file name matching, and the rules are similar to [POSIX fnmatch](https://linux.die.net/man/3 /fnmatch). Running `baulk search *` will list all packages.

In baulk, we can use `baulk list` to list all installed packages, and `baulk list pkgname` to list specific packages.

![](./docs/images/baulksearch.png)


### Miscellaneous

Baulk provides sha256sum b3sum two commands to help users calculate file hashes.


## Baulk Windows Terminal integration

Baulk also provides the `baulkterminal.exe` program, which is highly integrated with Windows Terminal and can start Windows Terminal after setting the Baulk environment variable, which solves the problem of avoiding conflicts caused by tool modification of system environment variables and anytime, anywhere Contradictions of loading related environment variables, in the compressed package distributed by Baulk, we added `script/installmenu.bat` `script/installmenu.ps1` script, you can modify the registry, add a right-click menu to open Windows Terminal anytime, anywhere.

![](./docs/images/menu.png)

## Baulk actuator

baulk provides the `baulk-exec` command, through which we can execute some commands with the baulk environment as the background. For example, `baulk-exec pwsh` can load the baulk environment and then start pwsh. This actually has the same effect as baulkterminal, but baulk-exec can solve scenarios where Windows Terminal cannot be used, such as in a container, when performing CI/CD.

## Baulk upgrade

At present, we use the Github Release Latest mechanism to upgrade Baulk itself. When executing Github Actions, when new tags are pushed, Github Actions will automatically create a release version and upload the binary compressed package. In this process, the tag information will be compiled into the baulk program. When running `baulk-update` locally (please note that baulk update is to update the bucket and baulk-update are not the same command), it will check whether the local baulk is in the tag, If it is not built on Github Actions, the next step will not be checked unless the `--force` parameter is set. If it is a tag built on Github Actions, check whether it is consistent with Github Release Latest, inconsistently download the binary of the corresponding platform, and then Update Baulk.

## Other

<div>Icons made by <a href="https://www.flaticon.com/authors/smashicons" title="Smashicons">Smashicons</a> from <a href="https://www.flaticon.com/" title="Flaticon">www.flaticon.com</a></div>