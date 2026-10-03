# skitter

**The `skitter` command: it writes an Android project, and then it drives one.**

```
skitter init myapp --id com.example.myapp --name "My App"
cd myapp
skitter run
```

`init` writes the project with your two names already in it and a git history of its own. `run`
builds it, installs it, launches it and follows the log.

## Written in sysl, and that is half the point

**This is the first program in the org that drives a whole toolchain** — git, curl, tar, Gradle,
adb — and it does it without a line of shell. That became possible in **sysl 0.0.78**, which added
[`sysl.process`](https://sysl.sh/library/process/); before it there was no way to start a child
process from sysl at all, so a tool like this could not have been written in the language it serves.

It depends on **nothing**. The argument parser, the filesystem, the environment, the text handling
and the process control are all `sysl.*`.

**No shell anywhere in it.** Every child is started by name with its arguments as a list, so nothing
here quotes anything: a path with a space in it is a path, and an application name with a `;` in it
is a name.

## The commands

| | |
|---|---|
| `skitter init <dir>` | write a new project — `--id` and `--name` are its two names |
| `skitter run` | build, install, launch, follow the log |
| `skitter build` | just the debug APK |
| `skitter install` | install what was last built |
| `skitter log` | follow a running application's output |
| `skitter release` | the release APK and the Play bundle, signed if you have a key |
| `skitter keygen` | make the release key, once |
| `skitter clean` | Gradle's output and sbt's |

## Publishing to Play

```
skitter keygen
skitter release
```

`keygen` makes an upload key in `~/.android/sysl-release.jks`, with a random password, and writes
`~/.android/sysl-signing.properties`, which is where the project's build looks for it. It refuses to
replace either file. **Back both up off the machine**: Play ties your listing to this key, and losing
it means you can no longer publish updates. `SYSL_SIGNING_PROPERTIES` names a different properties
file, for both `keygen` and the build. keytool is taken from the same JDK the build runs under.

`release` builds the APK and the bundle in one Gradle run, because they are one release. The APK is
for installing by hand and checking; the `.aab` is what Play takes. It then prints both paths, whether
they are signed, the version, and the `versionCode`. The version is `version` in
`program/package.hocon`, and the code is `MAJOR × 1000000 + MINOR × 1000 + PATCH`. Play refuses a
code that has not gone up, so bump the version before each upload.

The bundle holds `arm64-v8a`, which is the ABI Play requires from apps with native code, and Play
accepts a bundle with nothing else. Play also has a floor on the target API level, which rises every
August. The template targets 36, which meets it as of 2026. See the project's README, § Publishing
to Play.

## What `init` actually does

A **tarball**, not a clone. `sysl-lang/skitter-app` is fetched at a pinned tag and unpacked, so a new
project has no git history but its own — a clone would arrive carrying somebody else's commits and a
remote pointing at the template, which is the thing everyone has to remember to undo.

Then the two identity lines in `gradle.properties` (`skitter.applicationId`, `skitter.appName`) are
rewritten — the template's other settings, such as `skitter.sdlLibraries` and `skitter.permissions`,
are left at their defaults — the template's README and screenshot are
replaced with one about *your* project, and `git init` plus one commit leaves you somewhere sensible
to start from.

**The tag is pinned rather than tracking a branch**, so the project a given `skitter` writes is the
same one next month. A tool that scaffolded from whatever `dev` held would produce something
different on Tuesday than on Monday and neither would be reproducible.

## The two environment problems it solves

Both are here because each reports itself as something else entirely, and between them they account
for most of the time anybody loses to a first Android build.

- **Gradle runs on a JDK between 17 and 25 and on nothing else.** A newer one fails *inside* Gradle
  with a stack trace that never mentions a version. `skitter` finds one in range — asking
  `/usr/libexec/java_home` on macOS — and hands it to Gradle. **Whatever is already set wins if it
  works**, so somebody who has arranged their own toolchain is not overridden.
- **A missing Android SDK surfaces as a CMake toolchain error.** `ANDROID_HOME` is found, or the
  usual place is looked in.

Neither is exported into your shell: they are set for the child, between the fork and the exec, so
this program's own environment is untouched.

## What it needs

`git`, `curl` and `tar` for `init`; the Android SDK with the **NDK** and **CMake**, plus `sbt`, to
build. And a **sysl 0.0.78 or newer** to compile this program itself.

## Building it

```
sysl build .
```

## Licence

ISC.
