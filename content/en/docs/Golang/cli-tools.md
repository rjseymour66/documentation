---
title: "CLI tools"
weight: 80
description: >
  Working with CLI tools in Go.
---

## Flags

`flag.<FunctionName>` lets you define CLI flags. For example, to create a flag that performs an action if the flag is provided, you can use `flag.Bool`.

The following flag function definition returns the value of a `bool` variable that stores the value of the flag. After you create a flag, you have to call the `Parse()` function to parse the arguments provided to the command line:

```go
lines := flag.Bool("l", false, "Count the number of lines")
      // flag.Bool(flagName, default value, usage info)
flag.Parse()
```
> **IMPORTANT**: Each `flag.*` returns a pointer. To use the value in this variable that 'points' to an address, you have to derefence it with the `*` symbol. If you don't dereference, you will use the address of the variable, not the value stored at the address

Now, you have a variable `lines` that stores the address of a `bool` set to `false`. When a user includes the `-l` flag in the CLI invocation, `lines` is set to true. For `str := flag.String(...)`, the variable stores the string that the user enters after the `-str` flag.

### Multiple flags

If you use multiple flags in your application, use a `switch` statment to select the action based on the provided flags:

```go
switch {
case *flag1:
    // handle flag
case *flag2:
    // handle flag
default:
...
}
```
### Usage info

The default values for each flag are listed when the user uses the `-h` option. You can add a custom usage message with the `flag.Usage()` function. You have to assign `flag.Usage()` a immediately-executing function that prints info to STDOUT.

Place the `flag.Usage()` definition at the beginning of the main method:

```go
flag.Usage = func() {
    fmt.Fprintf(flag.CommandLine.Output(), "%s tool. Additional info\n", os.Args[0])
    fmt.Fprintf(flag.CommandLine.Output(), "Copyright 2022\n")
    fmt.Fprintln(flag.CommandLine.Output(), "Usage information:")
    // print the default settings for each flag
    flag.PrintDefaults()
}
```

```go
func cliFunc(r io.Reader, useLines bool) {}
cliFunc(os.Stdin, *lines)
```

### Reading flag arguments

When a flag accepts more than one arguments (such as a multiple strings), you can access each argument the `...` operator, similar to a variadic function.

The following function signature accepts an `io.Reader` and any arguments that follow the flag on the command line:

```go
t, err := getTask(os.Stdin, flag.Args()...) {
	// check error
}
```


----------------------------------------------------------------

## flag package

Go provides flag definition functions for common primitive types (`string`, `int`, etc.). A flag definition contains information about the flag such as defaults and usage information. The flag package parses command line flags with this flag definition.

The flag definition function can create internal variables for the flag and return a pointer to that variable, or it can use a variable that you define. For example, if you are defining a `string` flag, you use the `flag.String(...)` function to return a pointer to an internal variable, or you can use the `flag.StringVar(&userVar, ...)` to provide your own variable for the flag definition. `flag.*Var()` functions provide more control over variable definitions.

Each flag definition is saved in a structure called `*Flagset` for tracking. The flag package uses the `CommandLine` flag set when you define a flag.

The `Parse()` function extracts each command line flag in the `*Flagset` and creates name/value pairs, where the name is the flag name, and the value is the argument provided to the flag. Next, it updates any command line flag's internal variable.

### Changing flag usage type

You can replace the type that displays beside the flag in usage. In the usage string, enclose the replacement word in backticks (\`\`). For example, the following `flag.StringVar()` function accepts a `string` type by default. You can change that to a `URL` type with backticks:
```go
flag.StringVar(&f.url, "url", "", "HTTP server `URL` to make requests (required)")
```


### Manual implementation

```go
type flags struct {
	url  string
	n, c int
}

// parseFunc is a command-line flag parser function
type parseFunc func(string) error

func (f *flags) parse() (err error) {
	// map of flag names and parsers
	parsers := map[string]parseFunc{
		"url": f.urlVar(&f.url),
		"n":   f.intVar(&f.n),
		"c":   f.intVar(&f.c),
	}

	for _, arg := range os.Args[1:] {
		n, v, ok := strings.Cut(arg, "=")
		if !ok {
			continue // can't parse the flag
		}
		parse, ok := parsers[strings.TrimPrefix(n, "-")]
		if !ok {
			continue // can't find parser
		}
		if err := parse(v); err != nil {
			err = fmt.Errorf("invalid value %q for flag %s: %w", v, n, err)
			break
		}
	}
	return err
}

func (f *flags) urlVar(p *string) parseFunc {
	return func(s string) error {
		_, err := url.Parse(s)
		*p = s
		return err
	}
}

func (f *flags) intVar(p *int) parseFunc {
	return func(s string) (err error) {
		*p, err = strconv.Atoi(s)
		return err
	}
}
```

## Custom flag types

First, create a new type that satisfies the `Value` interface:
```go
type Value interface {
    Set(string)
    String() string
}
```

Next, register the type to the default flag set with `Var()`. Then, `Parse` can handle the flag.

## Positional arguments

Define `flag.Usage` as a function that prints usage text that is defined as a variable, and then the usage messages for the optional arguments:

```go
const usageText = `
Usage:
  hit [options] url
Options:`

...
func funcName() {
	flag.Usage = func() {
		fmt.Fprintln(os.Stderr, usageText[1:])
		flag.PrintDefaults()
	}

	flag.Var(toNumber(&f.n), "n", "Number of requests to make")
	flag.Var(toNumber(&f.c), "c", "Concurrency level")
	flag.Parse()

	f.url = flag.Arg(0)
    
    ...
}
```
In the previous example:
- `url` is the positional argument. It is included in the `usageText` constant.
- `flag.PrintDefaults()` method prints the usage information for the 
- `flag.Arg(0)` stores the first argument after the flag.

--------------------------------------------------------------

## Cobra CLI

TODO: Setup

1. Create the functions that the tool will use
2. Add the CLI option with `cobra-cli add <toolname>`
3. In `<toolname>.go`, update the fields in the &cobra.Command object as needed. Add some of the following fields, if necessary:
   - SilenceUsage:
   - Args: 
4. Update the `Run` field to `RunE`. `RunE` returns a function so you can test it. The signature returns an error.

### Install

Download and install from the [Github repo](https://github.com/spf13/cobra).

1. `$ go get -u github.com/spf13/cobra@latest`
2. `$ go install github.com/spf13/cobra-cli@latest`

### Start a project

Use the `cobra-cli` tool to init a project:
```bash
$ cobra-cli init <project-name>
```
Add subcommands to a project:
```bash
$ cobra add <subcommand-name>
```
This adds a new file with boilerplate code in the `/cmd` directory.

### Add subcommands

Cobra has a flag package is an alias to `pflag`, a replacement for Go's standard flag package that includes POSIX compliance.

Persistent flags use the following structure:

```go
rootCmd.PersistentFlags().StringP(<command-name>, <short-hand>, <default>, <short-desc>)

// example
rootCmd.PersistentFlags().StringP("hosts-file", "f", "pScan.hosts", "pScan hosts file")
```

Command to create a subcommand:

```shell
$ cobra-cli add <subcommand-name> -p <parent-command-instance-var>
```
The `instance variable` is the name of the command variable in `root.go`:

```go 

var hostsCmd = &cobra.Command{
	Use:   "hosts",
	Short: "Manage the hosts list",
	Long: "...",
}
```

For example:

```shell
$ cobra-cli add list -p hostsCmd
```

### Command completion and docs




### Viper

Install [Viper](https://github.com/spf13/viper):

```shell
$ go get github.com/spf13/viper
```

### Initial setup

```go
// cmd/root.go

func init() {
    ...

	// replace dash with underscore for some OSs
	replacer := strings.NewReplacer("-", "_")
	viper.SetEnvKeyReplacer(replacer)
	// add prefix to host file env var
	viper.SetEnvPrefix("PSCAN")

	// bind key to the flag
	viper.BindPFlag("hosts-file", rootCmd.PersistentFlags().Lookup("host-file"))

	...
}
```

### Persistent flags

Add these flags in the root.go file. Persistent flags are available to the command and all subcommands under that command.