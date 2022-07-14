# Installer Guidelines

## 0. Objectives

We want people to use Haskell, which means the path to installing Haskell for first-time users
should be short and easy, and it should yield an environment that is easy to use with the supporting
literature. In addition, building simple programmes with a fresh installation should be as quick
and painless as possible.

These guidelines will set out the criteria that the Haskell Foundation will use in evaluating
installers. We do not expect any individual to meet all these criteria initially but we would like
them to be met by a collection of installers as quickly as possible.

## 1. Cross Platform Coverage

As many platforms should be covered by the installers as possible. For now, the minimum platforms
that should be covered should include 64-bit Windows, macOS and Linux running on
Intel-x86-compatible processors. This set of minimal platforms will soon have to be extended to
include ARM-architecture processors.

## 2. Uniformity of Delivered Environment

Each installer should deliver a Haskell development environment that includes the following.

  * The current recommended GHC toolchain, as determined by the installer, should be installed out
    of the box, ready to use by all of the installed build systems.

  * Both of the principal build systems (`cabal-install` and `stack`) should be installed,
    available and ready for use in the way their associated development communities would expect.
    (If an installer interrogates the user on installation about which components to install it is
    fine to make `cabal-install` or `stack` optional, provided are both installed when the default
    options are take.)

  * It should be possible to install and manage multiple toolchains that both `stack` and
    `cabal-install` can use. This requires that both `stack` and `cabal-install` need to be
    configurable to access a GHC toolchain that has been installed by the other and the installer
    must configure both toolchains to work with the `starter` GHC toolchain.

## 3. User Installable

It should be possible to install Haskell in the user environment without recourse to administrator
privileges.

## 4. Editor/IDE Support

Although not an initial requirement, We would like to see support for installing important,
development infrastructure beyond the toolchains and the build systems, such as the Haskell Language
Server or the `haskell-mode` for an `emacs` installation.  The installer might not carry out any
such installation, but it could point the user at supporting documentation.

## 5. Preinstalled Packages

This won't be a minimum requirement to start with but the Haskell Foundation would like to encourage
Haskell installation systems to install a core set of precompiled packages, apart from `base`, when
installing the default compiler toolchain (at least). The objective is to allow developers
(especially new developers) to be able to build certain kinds of simple applications quickly and
painlessly after a fresh installation.

## 6. Minimal Requirements

Installer systems should not require as a prerequisite the installation of other major systems that
the average Haskell-curious developer would not be expected to have installed. The requirement that
`chocolatey` be already installed in order to install Haskell on Windows would be undesirable for
example.

Care should be taken to ensure that system libraries needed by the installation are installed or,
that the user gets clear instructions on how to install them. Developers should not encounter errors
as a matter of course while building basic applications.

## 7. Easy Upgrade and Removal Process

It should be easy to upgrade the installer and install new toolchains. It should also be easy to
remove any component from the systems, including the installer itself, the build system and the
toolchains.

## 8. No Aggressive Marketing

We would rather not see promotional material on the download page that suggests that any installer
system is the main/principal installer mechanism for Haskell.
