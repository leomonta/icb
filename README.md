# incremental C Builder

An incremental but simplistic build tool for personal projects

## Why:

When I started making stuff in C/C++ without an IDE (that compiles for you, VSCode does not) I encountered Makefile, and honestly, I hated it.
There are some cool stuff that you can do with it but is too complex and with way too many hidden and implicit rules that are just stupid (e.g. `.c.o`)

So my brain had the brillian idea of making an entire python script to compile projects the way i organize them. Cuz it was "Easier and Fun, Trust me"

I keep my files organized in specific directories, Source files in `src/`, includes in `include/`, object files (`.o`) in `obj/` and so on, Makefile make this hard for some reason (most probably I'm just too stupid / lazy to do it but nevermind).
Sometimes there are some variations from project to project, e.g. some times I use the `ext/` folder to keep libraries source files and includes, and often I use different libraries while linking

## Modus Operandis: (?)

A config file (`icb.json`) is usually needed. A `JSON` file that specify compiler, directories, lbraries and other compilation options.

Given the source directories (aka the directories containing source files) it attempts to compile all of the file recognized as source files (aka .c, .cpp .c++ ...) that have been modified since the last compilation
To know which files have been modified it computes an hash of the file itself and compares it to a saved copy of the previous compilation (the hashes are stored unceremoniously in `files_hash.txt`)
This check is also performed recursively for every `#include` the file contain. So, if an include file is changed (even at the system level) the modification is propagated through all the source files that use it. 

Different profiles are supported, they are just free floating keys in the root of the config file, that can be used via `-p profilename`.
For each profile a new subdirectory is created in the objects_path folder to contain the object files for that specific profile avoiding different profile to poison each other

Profiles are configured with 5 keys:
- compiler_args
- linker_args
- libraries_dirs
- libraries_names
- scripts
	- pre
	- post

Each of these key can be specified or not, if a key is not specified (a.k.a. not present), its value will be the default value.
Default values are empty strings for all of the keys, to overwrite the default value (for all profiles) you can specify a `default` profile.

Examples

```json

	...

	"pname": {
		"compiler_args": "-g3 -Wall ...",
		"linker_args": "-s -flto ...",
		"libraries_dirs": [
			"/path/to/library"
		],
		"libraries_names": [
			"pthread",
			"custom library"
		],
		"scripts": {
			"pre": "clean"
		}
	}

	...

```

This profile `pname`, even if not specified, has a `post` script equals to `""`

```json

	...

	"default": {
		"scripts": {
			"pre": "clean"
		}
	}

	"pname": {
		"compiler_args": "-g3 -Wall ...",
		"linker_args": "-s -ltco ...",
		"libraries_names": [
			"pthread",
			"raylib",
			"..."
		],
		"scripts": {
		}
	}

	...

```

This profile `pname` instead has a `post` script equals to `""` and a `pre` script equal to `clean`


To prevent inheriting default profile settings you can specify every key with an empty value

```json

	...

	"default": {
		"scripts": {
			"pre": "clean"
		}
	}

	"pname": {
		"compiler_args": "-g3 -Wall ...",
		"linker_args": "-s -ltco ...",
		"libraries_names": [
		],
		"libraries_dirs": [
			"pthread",
			"raylib",
			"..."
		],
		"scripts": {
			"pre" : "",
			"post" : ""
		}
	}

	...

```


## Process

Checks for cli switches

Loads the files hashes in an array
> If a complete compilation (`-a` switch) has been requested this step is skipped

Parse the config files and the profile, then saves the useful data in an internal dict
> The builder `cd`s in the `project_dir` so all the other dirs should be relative to that one

If the `pre` key is present in `scripts` execute the given script

Lists all of the files that are in the `source_dirs` and select only the one that can be compiled (e.g. .c, .cpp. .h) and have been modified
> Early exit if no files to compile are found

Creates a thread that calls the given compiler with all of the correct arguments for each file that needs to be compiled

When all the threads are done prints all of the compiler output
> Early exit if there is an error

Calls the given linker on all of the object files present in the profile directory and prints its output
> If a source file has been renamed or removed its old object file will remain in the object profile directory, an `rm` might be needed to prevent linking it with the final product

If the `post` key is present in `scripts` execute the given script

Saves the new hashes that have been generated
> If a complete compilation (`-a` switch) has been requested this step is skipped

## Makefile export

The makefile export might look a little 'quirky'

Firstly dumps the general (such as compiler and directories) in specific variables
Secondly it figures all the source files that will be used and all the object files that will be produced and puts them in their own variables

> even if `PROFILE` is empty it is only substituted when the rules are excetued

The `.SUFFIXES` is needed to prevent any implicit rule form firing

And the rule `$(SOURCES):` is needed to compile object files from the sources
> the object filename is obtained from the source filename

Each profile has its own variables, and for each profile there is a linking rule and a general rule.
The general rule calls, if present, any pre or post script at the required time (using the `|` after the `:`).
The linking rule instead overrides some global variables (`CARGS` and `PROFILE`) to tell the `$(SOURCES)` rule which args and which directory to use when compiling

The default profile is also added as a dummy in case it might be self sufficient, but its settings are propagated through all the other profiles

Lastly the `clean` rule is defined and all of the script rules are marked as `.PHONY`


## Known problems

NONE

Next..
jkjk

### Modified files are not identified properly

Yes they are, but not immediatly

When checking if a file has been modified or not the search stops at the first dependency that has been, in fact, modified.
This means that when `files_hash.txt` is empty, or simply a minimally complex amount of headers have been added, many calls to the builder are needed to reach the 'top' of the include chain

This is fixable but I'm probably not going to since it's not that big of a problem

## Options:

These are all of the options that can be passed to the builder

```

general options

	-a                    disregard the saved hash and rebuild the entire project
	-p <profile-name>     utilize the given profile specifies in the config file
	-e                    do not compile and export the `icb.json` as a Makefile
	--gen                 writes in the current directory an empty `icb.json` file
	-n <num-of-threads>   number of parallel threads to execute at the same time, default 12
	-h, --help            print this screen

printing options

	--skip-empty-reports  do not show reports that are empty
	--skip-warn-reports   do not show reports that contain only warnings
	--skip-all-reports    do not show reports

	--skip-progress       do not show the animations for compiling units
	--skip-statuses       do not show any status for compiling / done / failed compilations

	--no-colors           do not use colors for the output, same for compiler reports
```

## the icb.json structure

```json
{
	"compiler": {
		"compiler_style": "<what kind of compiler is being used (gcc, clang, msvc, rustc)>",
		"compiler_exe": "<path to the compiler executable>",
		"linker_exe": "<path to the linker executable>"
	},

	"directories": {

		"project_dir": "<project root directory relative to where the icb is being called>",
		"exe_path_name": "<path and name where to put the final executable>",
		"include_dirs": [
			"<additional include directories to pass to the compiler>"
		],
		"source_dirs": [
			"<directories where to search source files>"
		],
		"temp_dir": "<name of the directory where to put object files>"
	},

	"<profile name>": {
		"compiler_args": "<additional compiler args>",
		"linker_args": "<additional linker args>",
		"libraries_dirs": [
			"<additional libraries directories>"
		],
		"libraries_names": [
			"<additional libraries names>"
		],
		"scripts": {
			"pre": "<script to execute before the compilation begin>",
			"post": "<script to execute after the compilation end>"
		}
	}

}
```
