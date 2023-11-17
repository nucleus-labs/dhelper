
# DHelper

DHelper is a dev tool to help manage a development environment and automate configurable tasks.

Usage:
```
Main usage:
    ./dhelper [common-flag [flag-argument]]... <target> [target-flag [flag-argument]]... [target argument]...

Help aliases:
    ./dhelper
    ./dhelper  -h
    ./dhelper --help
    ./dhelper   help

More detailed help aliases:
    ./dhelper --help-target <target>
```

dhelper utilizes a hierarchical registration model. At the top-level, you have registration common to all targets. This includes flags, helper functions, etc. Some of these are built-ins provided by dhelper, all others are defined in `targets/common.bash`.


The core files for dhelper are as follows:
```
    ./
    ╠═ targets/
    ║   ╠═ common.bash
    ║   ╚═ <target>.bash
    ╠═ .env
    ╠═ arg_parse.bash
    ╠═ dhelper
    ║
    ╠═ .envrc
    ╠═ flake.nix
    ╚═ flake.lock
```

The 2nd group of files, `.envrc`, `flake.nix`, and `flake.lock` are provided for [nix flake](#) support.

## Flags

Flags are defined on two layers: common, and target-specific. The difference between them is that common flags are defined in `targets/common.bash` or as a builtin, and are parsed and executed before the target. Target-specific flags are defined in [target definitions](#targets). While they are defined in different locations depending on use-case, how they are defined is identical, and is as follows:
```bash
add_flag "-" "jobs" "sets the number of jobs/threads to use" 1 "job count" "int" "the number of jobs/threads to use"
function flag_name_jobs () {
    JOBS=$1
    [[ ! ${JOBS} =~ ^[0-9]+$ ]] && error $(eval echo "${ERR_INFO}") "[ERROR]: JOBS value '${JOBS}' is not a valid number!" 15
    debug "Using -j${JOBS}"
}
```

Let's break this down bit-by-bit:<br />
`add_flag "-" "jobs" "sets the number of jobs/threads to use" 1 "job count" "int" "the number of jobs/threads to use"` <br /><br />
`add_flag` is how you register a flag. This is irrespective of registration level. The arguments are as follows:
1. name-short           (char)
    - a single-character string that designates a short-flag alias. `-` designates a lack of short-flag alias. (`./dhelper -h` as opposed to `./dhelper --help`)
2. name-verbose         (string)
    - the long-form name of a flag. (eg. `help`: `./dhelper --help`)
3. description          (string)
    - a description of the flag's functionality
4. priority             (int)
    - an integer representing the order flags are run in. Flags are not immediately called upon parsing, instead having a deferred call. After all flags have been parsed, they are then sorted by priority and called from least to greatest.
5. argument-name        (string) (optional)
    - a string representing the name of an argument. This is purely for ease of developing and has no actual effect on the runtime other than help and error messages. If a flag has no arguments, you may provide no argument name (left empty).
6. argument-type        (string) (optional-dependent)
    - a string representing the variable's type. Valid types are (`"any" "string" "float" "int"`). `string` and `any` are identical in functionality, but `any` is intended to be an explicit "anything is accepted here". Type checking is performed on provided values that are not `any` or `string`, and will raise an error if the wrong type is provided. Providing a value for this is required if `argument-name` is provided, otherwise it is disallowed.
7. argument-description (string) (optional-dependent)
    - a description of what the argument does for the flag. Providing a value for this is required if `"argument-name"` is provided, otherwise it is disallowed.
<br /><br />

Next we have
```bash
function flag_name_jobs () {
```
This declares and defined the function called when the flag (as registered with `add_flag`) is used. This is pattern-matched using `flag_name_<name-verbose>`, where occurrences of a hyphen (`"-"`) are converted to underscores. So `--job-count` would be `"function flag_name_job_count () { ... }"`

The next step is `JOBS=$1`. A flag's argument is consumed from the command line input during parsing, and provided to the handler function during execution, so it's safe to directly use `"$1"`. However, there are no optional flag arguments, so keep this in mind when you're designing and implementing your flags.


## Targets

Targets (which can be thought of as subcommands) get a bit more complicated, being able to have any number of arguments and any number of their own flags. However, target registration is fairly similar to flag registration, with some key differences. The registration of a target starts with the creation of a file `targets/<target-name>.bash`. The presence of a file in the target directory that has the `.bash` extension is what defines a target, and is how target validation is performed. The first validation step checks that the target provided exists as a file in the `targets/` directory.

<br />
A target is defined within these files as such:<br />

`targets/dummy-types.bash`:
```bash
add_argument "dummy1" "int"     "a dummy int"
add_argument "dummy2" "float"   "a dummy float"
add_argument "dummy3" "string"  "a dummy string"

function target_dummy_types () {
    echo "dummy1: $1"
    echo "dummy2: $2"
    echo "dummy3: $3"
}
```

This follows the pattern of flag registration pretty closely. We start by adding arguments, but we can add multiple arguments. The parameters are as such:

1. argument-name        (string)
    - a string representing the name of an argument. This is purely for ease of developing and has no actual effect on the runtime other than help and error messages.
2. argument-type        (string)
    - a string representing the variable's type. Valid types are (`"any" "string" "float" "int"`). `string` and `any` are identical in functionality, but `any` is intended to be an explicit "anything is accepted here". Type checking is performed on provided values that are not `any` or `string`, and will raise an error if the wrong type is provided.
3. argument-description (string)
    - a description of what the argument does for the flag.

The target parameters are passed to the target handler function in the order that they are defined, so be sure to keep that in mind when implementing your targets.

Unlike flag arguments, none of these are optional since you're explicitly defining arguments.

### targets/common.bash

This is a unique target that is ignored by target validation and is specifically intended for definitions that are shared between targets. These may be variables, functions, what have you. Anything used between multiple targets can and should be defined here.

### .env

If a `.env` file does not exist when you run dhelper, it will be created. This will overwrite variable values from `arg_parse.bash` and `targets/common.bash`. Command-line flags will overwrite values defined in the `.env`. The precedent is as such: <br />
`arg_parse.bash` -> `targets/common.bash` -> `.env` -> `command-line flags`

### variadic target arguments
When providing a type for a target argument, the inclusion of an elipses (`...`) indicates that all following arguments (while being parsed) belong to this variable and are of this type. As such, they will be typechecked as such.

`targets/dummy-types.bash`:
```bash
add_argument "dummy1" "int"      "a dummy int"
add_argument "dummy2" "string"   "a dummy string"
add_argument "dummy3" "float..." "a dummy float"

function target_dummy_types () {
    echo "dummy1: $1"
    echo "dummy2: $2"

    shift 2 # discard the first 2 arguments

    # `$*` grabs all arguments passed to function, which is why you need to discard the first 2
    local dummy3=($*)
    IFS=','
    echo "dummy3: (${dummy3[*]})"
    unset IFS
}
```
usage:<br/>
`./dhelper dummy-types 5 "hello" 1.0 2.5 900.1`

output:
```
dummy1: 5
dummy2: hello
dummy3: (1.0,2.5,900.1)
```

