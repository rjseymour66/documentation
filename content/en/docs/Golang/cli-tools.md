---
title: "CLI tools"
weight: 80
description: >
  Working with command line interface (CLI) tools in Go.
---

## Flags

When you create flags in Go, each flag definition is saved in a structure called [*Flagset](https://pkg.go.dev/flag#FlagSet) for tracking. The default Flagset for the `flag` package is named `CommandLine`, and it has access to all Flagset functions.


The `Parse()` function extracts each command line flag in the `*Flagset` and creates name/value pairs, where the name is the flag name, and the value is the argument provided to the flag. Next, it updates any command line flag's internal variable.




Define flags and execute their logic in the main method. Think of CLI flag implementations as programs that call external libraries, even if the library is included in the same project.



A more complex and modular implementation consists of the following:
- A FlagSet, defined in a file separate from `main.go`
- Usage info

## Simple implementation

A simple implementation of the Go `flag` package consist of the following sections:
- Usage
- Flag definition
- A switch statement that evaluates the flags provided to the command line

### Usage

Create the usage information with the `Usage` variable, and place it at the beginning of the `main` method. `Usage` is a pointer to an immediately-executing function that prints messages about the Flagset to STDOUT. Your job is to create the custom function that `Usage` points to.

The following function defines a series of `Fprint[x]` statements that write formatted strings to the default Flagset, `Commandline`. `Commandline` uses its `Output` method to write to the usage destination. Finish the function definition with the `PrintDefaults()` method to print usage information for each flag:

You can create a `const` multi-line string and write it to `flag.CommandLine.Output()`. Make sure you slice the string after the first index to remove the leading newline character:

```go
const usage = `
<toolname>
Copyright 2023
Usage:
  todo [options]
Options:`

func main() {
	flag.Usage = func() {
		fmt.Fprintln(flag.CommandLine.Output(), usage[1:])
		flag.PrintDefaults()
	}
```

### Flag definition

After the `Usage` function, create the flag definitions. Go provides flag definition functions for common primitive types (`string`, `int`, etc.). A flag definition contains information about the flag, such as defaults and usage information.

The flag definition function can create internal variables for the flag in the Flagset and return a pointer to that variable, or it can use a variable that you define. For example, if you are defining a `string` flag, you use the `flag.String(...)` function to return a pointer to an internal variable, or you can use the `flag.StringVar(&varName, ...)` to provide your own variable for the flag definition. `flag.*Var()` functions provide more control over variable definitions.

Below are examples of both flag definition types. After you define all flags, you must call the `Parse()` function to parse the arguments provided to the command line:

```go
var lang string
flag.StringVar(&lang, "lang", "en", "The required language, e.g. en, ur...")

lines := flag.Bool("l", false, "Count the number of lines")
lang := flag.String("lang", "en", "The required...")

flag.Parse()
```
Now, you have a variable `lines` that stores the address of a `bool` set to `false`. When a user includes the `-l` flag in the CLI invocation, `lines` is set to true. For `lang := flag.String(...)`, the variable stores the string that the user enters after the `-lang` flag.

You do not have to define a `help` flag--Go provides the `-h` flag by default.

### Switch statement

When it's time to evaluate a Flagset that contains more than one flag--after you have checked for any environment variables or completed any other logic--use a `switch` statement.

> **IMPORTANT**: Each `flag.[Type]` flag definition function returns a pointer. To use the value in this variable that 'points' to an address, you have to derefence it with the `*` symbol. If you don't dereference, you will use the address of the variable, not the value stored at the address

Each `case` statement should handle a flag defintion. You can evaluate flag types such as `int` or `string` with an expression. When you evaluate a `Bool` flag, check that it is set to true. (By default, a `Boolean` flag is set to false. When a user includes the flag, it is set to `true`.) The `default` case should print usage information and exit. For example:

```go
switch {
case  iFlag > 0:
    // handle flag
case *boolFlag:
    // handle flag
default:
	// Invalid flag provided
	flag.Usage()
	os.Exit(1)
}
```
## Test the simple implementation

## Reading flag arguments

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