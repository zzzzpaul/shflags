# Introduction #
This is the documentation for the shFlags 1.0.x release series.

At its core, shFlags is simply a library that you include into an existing shell script that gives you some additional functions that can be called. The power behind those functions though is somewhat amazing though, and you will hopefully be amazed with the simplicity with which you can handle command-line arguments in shell.

# Table of Contents #
  * [Quick Start](#Quick_Start.md)
  * [Usage](#Usage.md)
    * [Understanding flag parsing](#Understanding_flag_parsing.md)
    * [Supported flag types](#Supported_flag_types.md)
    * [Automated help](#Automated_help.md)
  * [Examples](#Examples.md)
    * [Debug mode output](#Debug_mode_output.md)
    * [Accept flags and options](#Accept_flags_and_options.md)
  * [Known Issues](#Known_Issues.md)

# Quick Start #
Getting started with shFlags is quite easy. We'll start with the proverbial "Hello, world" application.

Just a quick note before we start. All examples assume that the `shflags` library has been copied the current directory, and that the scripts listed here are also in the current directory. You are free to move the `shflags` library wherever you want as long as you remember to update the line that sources the library.

Example: `examples/hello_world.sh`
```
#!/bin/sh

# source shflags
. /path/to/shflags

# define a 'name' command-line string flag
DEFINE_string 'name' 'world' 'name to say hello to' 'n'

# parse the command-line
FLAGS "$@" || exit $?
eval set -- "${FLAGS_ARGV}"

# say Hello!
echo "Hello, ${FLAGS_name}!"
```
_Don't forget to make the script executable with `chmod +x`._

Go ahead and give the script a run.
```
$ ./hello_world.sh 
Hello, world!
```

So what just happened? Let's go through the script line-by-line.

```
# source shflags
. /path/to/shflags
```
This line sources the `shflags` library (which is located in the fictitious `/path/to` directory) into the current shell environment. It brings in several functions and sets several variables, but you won't actually see any output from this line.

```
# define a 'name' command-line string flag
DEFINE_string 'name' 'world' 'name to say hello to' 'n'
```
Here we have defined a 'string' flag called `name`, we gave it a default value of 'world', we described the variable with 'name to say hello to', and we defined a short flag called `n`. This function has just told shFlags everything it needs to know about the `name` flag. It now knows:
  * how to handle the short command-line option `-n`
  * how to handle the long command-line option `--name Kate` (not supported on all systems)
  * how to describe the flag if `-h` (or `--help`) is given as an option
  * to accept and validate any input as a string value
  * to store any input (or the default value) in the `FLAGS_name` variable
Wow. That's a lot. Let's keep going.

```
# parse the command-line
FLAGS "$@" || exit $?
eval set -- "${FLAGS_ARGV}"
```
Here is where all the magic happens. The call to `FLAGS` with the passing of `"$@"` sends all command-line arguments into the shFlags library for processing. If everything is successful, a return value of 0 (`${FLAGS_TRUE}`) is returned an things continue. If a false value of 1 (`${FLAGS_FALSE}`) or an error value of 2 (`${FLAGS_ERROR}`) is returned, the processing is halted, returning that value (the `exit $?`). If there are any command-line arguments left over that shFlags didn't know how to process, they are now present in the **now updated** `$@` variable, which can be used as with any other script.

There is actually a lot that happened with this `FLAGS` call, but the basic action you will be interested in right now is that a `FLAGS_name` variable was defined. As we didn't pass the `-n` flag (or `--name`) on the command-line, the value placed in the variable is 'world', the default value specified when we defined the flag.

```
# say Hello!
echo "Hello, ${FLAGS_name}!"
```
Now that we have accepted parsed all our flags, we can make use of the results. This snippet prints out Hello to the screen ('Hello, world!' in this case) as the `FLAGS_name` variable contained 'world'.

Let's give the script another go, this time passing '`-n Kate`' on the command-line.
```
$ ./hello_world.sh -n Kate
Hello, Kate!
```
This time around, the `FLAGS_name` variable was defined with the value 'Kate' as we passed the `-n` flag, and 'Hello, Kate!' was output. I bet you never thought accepting command-line arguments could be so easy!

What about spaces in the flag value? Give it a try! _(Note: this requires the enhanced version of **`getopt`** at this time)_
```
$ ./hello_world.sh --name 'Kate Ward'
Hello, Kate Ward!
```
It works!

What happens if you can't remember the command-line flags you have defined, and what to find that out? Try passing the `-h` flag (or `--help`).
```
$ ./hello_world.sh -h
USAGE: ./hello_world.sh [flags] args
flags:
  -h  show this help
  -n  name to say hello to
```
There you have it. A list of all the command line options supported in a nice clean format. What could be easier?

# Usage #
Until this section gets written, see the source code in `src/shflags` and the various unit tests.

## Understanding flag parsing ##
The actual flag parsing step looks deceptively simple, but at the same time confusing.

```
FLAGS "$@" || exit $?
eval set -- "${FLAGS_ARGV}"
```

The `FLAGS "$@" || exit $?` takes the command-line arguments `$@` in their unparsed form and passes them to the `FLAGS` function for processing. If `$*` had been used, any spaces in string arguments that were passed would be lost. The double-quotes surrounding the `$@` are also required to maintain the spaces in string arguments. If `FLAGS` returns with a failure (represented by ${FLAGS\_ERROR}) the script is exited with an exit code of 1.

The `eval set -- "${FLAGS_ARGV}` replaces the $1, $2, $@, etc. variables with the non-flag values so that your script can process them normally. This is typically useful when you want to pass some flags to your script to alter processing, but you still want to allow non-flag type options to be passed as well.

## Supported flag types ##
shFlags supports the following flag types. As you may already know, shell does not natively support some of these types (e.g. float) so their actual implementation and usage will be a string.

### boolean ###
Boolean values are true/false values or conditions. A typical use might be to have support for some special functionality to be disabled, but a boolean flag could enable that functionality (e.g. enabling debug mode output).

Boolean flags do not take any arguments, and their value is either 1 (false) or 0 (true). For long flags, the false value is specified on the command line by prepending the word 'no'. With short flags, the presense of the flag toggles the current value between true and false. Specifying a short boolean flag twice on the command results in returning the value back to the default value.

In your script, you can use the `${FLAGS_TRUE}` or `${FLAGS_FALSE}` constants to compare the value of your flag variable against the "official" true/false values. Don't forget that in shell, true is 0 and false is 1 (quite the opposite of normal programming languages where true is 1 and false is 0).

A default value is required for boolean flags.

For example, lets say a Boolean flag was created whose long name was 'update' and whose short name was 'x', and the default value was 'false'. This flag could be explicitly set to 'true' with '--update' or by '-x', and it could be explicitly set to 'false' with '--noupdate'.

### float ###
Float values are signed numbers that may or not contain a decimal point.

As shell does not support float values natively, they are represented internally as strings. As such, any comparisons must be done as string comparisons using the = and != operators – `eq`, `ge`, `gt`, `le`, `lt`, and `ne` will not work.

### integer ###
Float values are signed numbers that do not contain a decimal point.

Shell supports integer values natively, so standard integer operators (`eq`, `ge`, `gt`, ...) may be used for comparisons.

### string ###
Strings are, well, strings. Treat them like you would any other string.

## Automated help ##
Unlike `getopt`, shFlags provides an automated means of generating help for the user. It has its limitations, but it is definitely better than nothing.

To get access to help, simply add the `-h` (or `--help`) flag to the command line, and you will get help for all of the defined flags.

If you want to make your usage line a bit more informative, perhaps even with a multiline string of text, simply define the `FLAGS_HELP` variable and shFlags will make use of it.

# Examples #
Most scripts in this section are also provided in the `examples` directory. If you cannot find the script in the released version, please check the checked in source.

**NOTE:** All scripts in this section expect that `shflags` exists in the current directory. If it doesn't, the line sourcing the `shflags` library must be modified.

## Debug mode output ##
This script sets up a `debug` flag that when specified will enable debug output to `STDERR`. If the flag is not specified (the default behavior), debug output is not enabled.

Example: `examples/debug_output.sh`
```
#!/bin/sh

# source shflags library from the current directory
. ./shflags

# define flags
DEFINE_boolean 'debug' false 'enable debug mode' 'd'

debug() {
  [ ${FLAGS_debug} -eq ${FLAGS_TRUE} ] && echo "DEBUG: $@" >&2
}

main()
{
  debug 'debug mode enabled'
  echo 'something interesting'
}

# parse the command-line
FLAGS "$@" || exit $?
eval set -- "${FLAGS_ARGV}"
main "$@"
```

## Accept flags and options ##
This script will try to write the current date to a filename given on the command-line. If the file already exists, the script will fail, but if a `-f` (or `--force`) flag is given, the existing file will be overwritten.

Example: `examples/write_date.sh`
```
#!/bin/sh

# source shflags
. ./shflags

write_date() { date >"$1"; }

# configure shflags
DEFINE_boolean 'force' false 'force overwriting' 'f'
FLAGS_HELP="USAGE: $0 [flags] filename"

main()
{
  # check for filename
  if [ $# -eq 0 ]; then
    echo 'error: filename missing' >&2
    flags_help
    exit 1
  fi
  filename=$1

  if [ ! -f "${filename}" ]; then
    write_date "${filename}"
  else
    if [ ${FLAGS_force} -eq ${FLAGS_TRUE} ]; then
      write_date "${filename}"
    else
      echo 'warning: filename exists; not overwriting' >&2
      exit 2
    fi
  fi
}

# parse the command-line
FLAGS "$@" || exit $?
eval set -- "${FLAGS_ARGV}"
main "$@"
```

A test without a filename should fail (notice the custom usage definition line).
```
$ ./write_date.sh
error: filename missing
USAGE: ./write_date.sh [flags] filename
flags:
  -h  show this help
  -f  force overwriting
$ echo $?
1
```

The first run should create the file just fine.
```
$ ./write_date.sh junk.dat
$ cat junk.dat 
Thu Jun 26 23:06:34 IST 2008
```

The second run should fail with an error.
```
$ ./write_date.sh junk.dat
warning: filename exists; not overwriting
$ echo $?
2
```

The third run succeeds because we pass the `-f` flag.
```
$ ./write_date.sh -f junk.dat
$ cat junk.dat 
Thu Jun 26 23:10:02 IST 2008
```

# Known Issues #
See the appropriate `RELEASE_NOTES` for the most up-to-date list of known issues.

The `getopt` version provided by default with all versions of Mac OS X (up to and including 10.8) and Solaris (up to and including Solaris 10 and OpenSolaris) is the standard version.