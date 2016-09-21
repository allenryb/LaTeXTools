# Alternative Builders

## Basic Builder

The basic builder is a simple, straight-forward build system. that simply runs the configured build engine (pdflatex, xelatex, or lualatex) and bibtex or biber if necessary. It can also be configured to support bibtex8 through the `bibtex` builder setting. In addition, it supports the [TeX Options](#tex-options) feature, the [output and auxiliary directory](#output-directory-and-auxiliary-directory) features and the [Jobname](#jobname) feature. It has been included because the default builder on MiKTeX, `texify` cannot be easily coerced to support biber or any of the other features supported by the basic builder. Note, however, that unlike `texify`, the basic builder does **not** support `makeindex` and friends (patches are welcome!).

You can use the basic builder by changing the `builder` setting to `"basic"`. It will read the same settings as the traditional builder.

## Script Builder

LaTeXTools now supports the long-awaited script builder. It has two primary goals: first, to support customization of simple build workflows and second, to enable LaTeXTools to integrate with external build systems in some fashion.

Note that the script builder should be considered an advanced feature. Unlike the "traditional" builder it is not designed to "just work," and is not recommend for those new to using TeX and friends. You are responsible for making sure your setup works. Please read this section carefully before using the script builder.

For the most part, the script builder works as described in the [Compiling LaTeX files](#compiling-latex-files) section *except that* instead of invoking either `texify` or `latexmk`, it invokes a user-defined series of commands. Note that although the Script Builder supports **Multi-file documents**, it does not support either the engine selection or passing other options via the `%!TEX` macros.

The script builder is controlled through two settings in the *platform-specific* part of the `builder_settings` section of `LaTeXTools.sublime-settings`, or of the current project file (if any):

- `script_commands` — the command or list of commands to run. This setting **must** have a value or you will get an error message.
- `env` — a dictionary defining any environment variables to be set for the environment the command is run in.

The `script_commands` setting should be either a string or a list. If it is a string, it represents a single command to be executed. If it is a list, it should be either a list of strings representing single commands or a list of lists, though the two may be mixed. For example:

```json
{
	"builder_settings": {
		"osx": {
			"script_commands":
				"pdflatex -synctex=1 -interaction=nonstopmode"
		}
	}
}
```

Will simply run `pdflatex` against the master document, as will:

```json
{
	"builder_settings": {
		"osx": {
			"script_commands":
				["pdflatex -synctex=1 -interaction=nonstopmode"]
		}
	}
}
```

Or:

```json
{
	"builder_settings": {
		"osx": {
			"script_commands": 
				[["pdflatex", "-synctex=1 -interaction=nonstopmode"]]
		}
	}
}
```

More interestingly, the main list can be used to supply a series of commands. For example, to use the simple pdflatex -> bibtex -> pdflatex -> pdflatex series, you can use the following settings:

```json
{
	"builder_settings": {
		"osx": {
			"script_commands": [
				"pdflatex -synctex=1 -interaction=nonstopmode",
				"bibtex",
				"pdflatex -synctex=1 -interaction=nonstopmode",
				"pdflatex -synctex=1 -interaction=nonstopmode"
			]
		}
	}
}
```

Note, however, that the script builder is quite unintelligent in handling such cases. It will not note any failures nor only execute the rest of the sequence if required. It will simply continue to execute commands until it hits the end of the chain of commands. This means, in the above example, it will run `bibtex` regardless of whether there are any citations.

It is especially important to ensure that, in case of errors, TeX and friends do not stop for user input. For example, if you use `pdflatex` on either TeXLive or MikTeX, pass the `-interaction=nonstopmode` option. 

Each command can use the following variables which will be expanded before it is executed:

|Variable|Description|
|-----------------|------------------------------------------------------------|
|`$file`   | The full path to the main file, e.g., _C:\\Files\\document.tex_|
|`$file_name`| The name of the main file, e.g., _document.tex_|
|`$file_ext`| The extension portion of the main file, e.g., _tex_|
|`$file_base_name`| The name portion of the main file without the, e.g., _document_|
|`$file_path`| The directory of the main file, e.g., _C:\\Files_|
|`$aux_directory`| The auxiliary directory set via a `%!TEX` directive or the settings|
|`$output_directory`| The output directory set via a `%!TEX` directive or the settings|
|`$jobname`| The jobname set via a `%!TEX` directive or the settings|

For example:

```json
{
	"builder_settings": {
		"osx": {
			"script_commands": [[
				"pdflatex", 
				"-synctex=1"
				"-interaction=nonstopmode",
				"$file_base_name"
			]]
		}
	}
}
```

Note that if none of these variables occur in the command string, the `$file_base_name` will be appended to the end of the command. This may mean that a wrapper script is needed if, for example, using `make`.

Commands are executed in the same path as `$file_path`, i.e. the folder containing the main document. Note, however, on Windows, since commands are launched using `cmd.exe`, you need to be careful if your root document is opened via a UNC path (this doesn't apply if you are simply using a mapped drive). `cmd.exe` doesn't support having the current working directory set to a UNC path and will change the path to `%SYSTEMROOT%`. In such a case, just ensure all the paths you specify are absolute paths and use `pushd` in place of `cd`, as this will create a (temporary) drive mapping.

### Supporting output and auxiliary directories

If you are using LaTeXTools output and auxiliary directory behavior there are some caveats to be aware of. First, it is, of course, your responsibility to ensure that the approrpiate variables are passed to the appropriate commands in your script. Second, `pdflatex` and friends do not create output directories as needed. Therefore, at the very least, your script must start with either `"mkdir $output_directory"` (Windows) or `"mkdir -p $output_directory"` and a corresponding command if using a separate `$aux_directory`. Note that if you `\include` (or otherwise attempt anything that will `\@openout` a file in a subfolder), you will need to ensure the subfolder exists. Otherwise, your run of `pdflatex` will fail.

Finally, unlike Biber, bibtex (and bibtex8) does not support an output directory parameter, which can make it difficult to use if you are using the LaTeXTools output directory behavior. The following work-arounds can be used to get BibTeX to do the right thing.

On Windows, run BibTeX like so:

```
cd $aux_directory & set BIBINPUTS=\"$file_path:%BIBINPUTS%\" & bibtex $file_base_name
```

And on OS X or Linux, use this:

```
"cd $output_directory; BIBINPUTS=\"$file_path;$BIBINPUTS\" bibtex $file_base_name"
```

In either case, these run bibtex *inside* the output / auxiliary directory while making the directory containing your main file available to the `BIBINPUTS` environment variable. Note if you use a custom style file in the same directory, you will need to apply a similar work-around for the `BSTINPUTS` environment variable.

### Supporting jobname

If you are using LaTeXTools jobname behaviour, you should be aware that you are responsible for ensure jobname is set in the appropriate context. In particular, a standard build cycle might look something like this:

```json
"builder_settings": {
	"osx": {
		"script_commands": [
			"pdflatex -synctex=1 -interaction=nonstopmode -jobname=$jobname $file_base_name",
			"bibtex $jobname",
			"pdflatex -synctex=1 -interaction=nonstopmode -jobname=$jobname $file_base_name",
			"pdflatex -synctex=1 -interaction=nonstopmode -jobname=$jobname $file_base_name"
		]
	}
}
```

### Caveats

LaTeXTools makes some assumptions that should be adhered to or else things won't work as expected:
- the final product is a PDF which will be written to the output directory or the same directory as the main file and named `$file_base_name.pdf`
- the LaTeX log will be written to the output directory or the same directory as the main file and named `$file_base_name.log`
- if you change the `PATH` in the environment (by using the `env` setting), you need to ensure that the `PATH` is still sane, e.g., that it contains the path for the TeX executables and other command line resources that may be necessary.

In addition, to ensure that forward and backward sync work, you need to ensure that the `-synctex=1` flag is set for your latex command. Again, don't forget the `-interaction=nonstopmode` flag (or whatever is needed for your tex programs not to expect user input in case of error).

Finally, please remember that script commands on Windows are run using `cmd.exe` which means that if your script uses any UNC paths will have to use `pushd` and `popd` to properly map and unmap a network drive.

## Customizing the Build System

Since the release on March 13, 2014 ([v3.1.0](https://github.com/SublimeText/LaTeXTools/tree/v3.1.0)), LaTeXTools has had support for custom build systems, in addition to the default build system, called the "traditional" builder. Details on how to customize the traditional builder are documented above. If neither the traditional builder nor the script builder meet your needs you can also create a completely custom builder which should be able to support just about anything you can imagine. Let me know if you are interested in writing a custom builder!

Custom builders are small Python scripts that interact with the LaTeXTools build system. In order to write a basic builder it is a good idea to have some basic familiarity with the [Python language](https://www.python.org/). Python aims to be easy to understand, but to get started, you could refer either to the [Python tutorial](https://docs.python.org/3/tutorial/index.html) or any of the resources Python suggests for [non-programmers](https://wiki.python.org/moin/BeginnersGuide/NonProgrammers) or [those familiar with other programming languages](https://wiki.python.org/moin/BeginnersGuide/Programmers).

LaTeXTools comes packaged with a small sample builder to demonstrate the basics of the builder system, called [`SimpleBuilder`](https://github.com/SublimeText/LaTeXTools/blob/master/builders/simpleBuilder.py) which can be used as a reference for what builders can do.

If you are interested in developing your own builder, please see [our page on the wiki](https://github.com/SublimeText/LaTeXTools/wiki/Custom-Builders) with documentation and code samples!