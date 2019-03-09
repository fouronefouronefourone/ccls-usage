# CCLS

## Building CCLS

Aside from Homebrew (which I find a pain to use), there isn't any easy installation for `ccls` on macOS/OSX. As such I just build and install from source.

The source for `ccls` lives at `https://github.com/MaskRay/ccls`

To build `ccls`, you must have `cmake` and the `clang+llvm-7.0.0` headers from `release.llvm.org` visible to `cmake`.

Here's a quick-n-dirty bash script for getting the headers and building a release version of `ccls`

```sh
mkdir clang-llvm
cd clang-llvm

wget "http://release.llvm.org/7.0.0/clang+llvm-7.0.0-x86_64-apple-darwin.tar.xz"
tar -xvf clang+llvm-7.0.0-x86_64-apple-darwin.tar.xz

cd ..


cmake -H. -BRelease -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=`pwd`/clang-llvm/clang+llvm-7.0.0-x86_64-apple-darwin
cmake --build Release
```

## Integrating CCLS with Sublime Text

I use the `LSP` plugin (`https://github.com/tomb564/LSP`) to integrate ccls into Sublime Text:

`LSP` doesn't support `ccls` by default as a language server. However, it is easy to add support for `ccls` by modifying `LSP.sublime-settings` like so:

```json
// Settings in here override those in "LSP/LSP.sublime-settings",
{
    "complete_using_text_edit": true,
    "clients": {
        "command": [
            "ccls"
        ],
        "initializationOptions":
        {
            "cacheDirectory": "/tmp/ccls"
        },
        "languages":
        [
            {
                "languageId": "c",
                "scopes":
                [
                    "source.c"
                ],
                "syntaxes":
                [
                    "Packages/C++/C.sublime-syntax"
                ]
            },
            {
                "languageId": "cpp",
                "scopes":
                [
                    "source.c++"
                ],
                "syntaxes":
                [
                    "Packages/C++/C++.sublime-syntax"
                ]
            },
            {
                "languageId": "objective-c",
                "scopes":
                [
                    "source.objc"
                ],
                "syntaxes":
                [
                    "Packages/Objective-C/Objective-C.sublime-syntax"
                ]
            },
            {
                "languageId": "objective-cpp",
                "scopes":
                [
                    "source.objc++"
                ],
                "syntaxes":
                [
                    "Packages/Objective-C/Objective-C++.sublime-syntax"
                ]
            }
        ]
    }
}
```

## Finalising CCLS against a codebase

`ccls` differs from `ctags` or `cscope` in that it needs a little more prepwork in order to have an indexing database for a codebase. In particular, a file called `compile_commands.json` must exist. `compile_commands.json` is a `.json` file that contains a list of commands that are used to compile each source file.

How `compile_commands.json` is generated depends on the codebase, for some codebases (like jsc) it can be a pain in the ass to get a usable `compile_commands.json` file.

Misc note: `compile_commands.json` must be **located at a codebase root folder** in order for `ccls` to properly index.

Below i'll show you how to generate `compile_commands.json` for `v8` and `jsc`

### Setting up CCLS for V8

The ability to generate `compile_commands.json` is built into `ninja`, and as such generating it is straightforward for `v8`:

```sh
# Assuming you have a clean checkout of v8, and depot_tools is installed and visible

# in the root v8/ dir
tools/dev/v8gen.py x64.release

ninja -C out.gn/x64.release -t compdb cxx cc > compile_commands.json

# now generate the ccls database
ccls -index=. -log-file="log.txt" -v=3
```

### Setting up CCLS for JSC

As far as I can tell, there isn't any inbuilt ability for jsc to generate `compile_commands.json` via `Xcode`. As such, we use `xcpretty` (`https://github.com/xcpretty/xcpretty`) to generate `compile_commands.json`:

```sh
# in the root webkit/ dir

# NOTE: ensure that you are running on a clean checkout of webkit (or have run Tools/Scripts/clean-webkit)
#  otherwise compile_commands.json will not be properly generated
Tools/Scripts/build-jsc | xcpretty -r json-compilation-database -o compile_commands.json
```

However, this is not the end, as the generated `compile_commands.json` as it stands does not contain the full build log. This is because during the compilation process, webkit combines common `.cpp` files into "unified sources", and then runs compilation commands on those unified sources instead of the original files.

To get around this, I use this simple script to unpack unified source files back into a format that `ccls` is happy with:

```python
#!/usr/bin/env pypy3

# NOTE: requires the pip package 'funcy'

import argparse
import unittest

from re import sub
from pathlib import Path
from json import loads, dumps
from funcy import re_all, lfilter
from unittest import TextTestRunner, TestLoader, TestSuite


# Tools/Scripts/build-jsc | xcpretty -r json-compilation-database -o compile_commands.json


class TestUnpacker(unittest.TestCase):
    def setUp(self):
        self.base_json = load_json(Path("./testdata/base-compile_commands.json"))
        self.refined_json = load_json(Path("./testdata/refined-compile_commands.json"))
        self.maxDiff = None

    def test_items(self):
        formatter = WebkitCompileCommandFormatter("../webkit")
        refined_json = formatter.refine_json(self.base_json)

        for cmd in self.refined_json:
            cmd_name = Path(cmd["file"]).name.strip()
            self.assertFalse(cmd_name.startswith("UnifiedSource"))
            # TODO: some more sanity checks for returned data

    def test_full(self):
        formatter = WebkitCompileCommandFormatter("../webkit")
        refined_json = formatter.refine_json(self.base_json)

        self.assertListEqual(refined_json, self.refined_json)


def parse_args():
    argp = argparse.ArgumentParser()

    argp.add_argument(
        "-f", "--compile-commands-path",
        dest="compile_commands_path",
        default="./compile_commands.json",
        help=".json file of compile commands to unpack",
        type=lambda x: Path(str(x))
    )

    argp.add_argument(
        "-dir", "--webkit-source-dir",
        dest="webkit_dir",
        default="../webkit",
        help="relative path of webkit source directory",
        type=lambda x: Path(str(x))
    )

    argp.add_argument(
        "--run-tests",
        dest="run_tests",
        help="run testsuite and exit",
        # action="store_false",
        action="store_true",
    )

    return argp.parse_args()


def load_json(fpath):
    return loads(load_file(fpath))


def load_file(fpath):
    return Path(fpath).read_text(encoding="utf-8")


class PackedBuildCommand(object):
    @property
    def is_unified(self):
        return self.file_name.startswith("UnifiedSource")

    @property
    def is_cpp(self):
        return self.file_suff in {".cpp", ".c", ".h"}

    @staticmethod
    def unpack_unified_source(base_dir, compile_cmd):
        command = compile_cmd["command"]
        directory = compile_cmd["directory"]

        raw_text = load_file(compile_cmd["file"])

        def decompress_cmd(in_filename):
            actual_path = Path(base_dir, in_filename).resolve()

            return {
                "command": sub(compile_cmd["file"], str(actual_path), command),
                "directory": directory,
                "file": str(actual_path)
            }

        return lfilter(lambda x: x, [decompress_cmd(x) for x in re_all(r"#include \"(.*)\"", raw_text)])

    def cmds_as_unpacked(self, webkit_dir):
        if not self.is_unified:
            self.cmd_json["file"] = str(Path(webkit_dir, self.cmd_json["file"]).resolve())

            fpath = self.cmd_json["file"]
            # print(f"--> path: {fpath}")

            return self.cmd_json

        return self.unpack_unified_source(webkit_dir, self.cmd_json)

    def __init__(self, cmd_json):
        self.cmd_json = cmd_json

        self.file_name = Path(cmd_json["file"]).name.strip()
        self.file_suff = Path(cmd_json["file"]).suffix


class WebkitCompileCommandFormatter(object):
    def __init__(self, webkit_dir):
        self.webkit_dir = Path(webkit_dir)
        self.webkit_dir_resolved = self.webkit_dir.resolve()
        assert(self.webkit_dir.is_dir())

        self.webkit_cmd_json = self.webkit_dir.joinpath("compile_commands.json")

    def refine_json(self, raw_json):
        refined_cmds = []

        for packed in [PackedBuildCommand(x) for x in raw_json]:
            unpacked = packed.cmds_as_unpacked(self.webkit_dir_resolved)
            if isinstance(unpacked, list):
                refined_cmds += unpacked
            else:
                refined_cmds.append(unpacked)

        return refined_cmds


def main(args):
    if args.run_tests is True:
        tests_to_run = TestSuite([x for x in TestLoader().loadTestsFromTestCase(TestUnpacker)])
        TextTestRunner(verbosity=2).run(tests_to_run)
    else:
        print(f"[x] loading JSON from '{args.compile_commands_path}'")
        cmds_json = load_json(args.compile_commands_path)

        formatter = WebkitCompileCommandFormatter(args.webkit_dir)
        refined_json = formatter.refine_json(cmds_json)

        # test against good json
        print(f"[i] wrote updated '{args.compile_commands_path}' -> '{formatter.webkit_cmd_json}'")
        Path(formatter.webkit_cmd_json).write_text(dumps(refined_json), encoding="utf-8")
        print("[x] finished goodbye")


if __name__ == '__main__':
    main(parse_args())
```

We use the above python script to generate a usable `compile_commands.json` for webkit like so:

```sh
# in the root webkit/ dir

# NOTE: ensure that you are running on a clean checkout of webkit (or have run Tools/Scripts/clean-webkit)
#  otherwise compile_commands.json will not be properly generated
Tools/Scripts/build-jsc | xcpretty -r json-compilation-database -o raw-compile_commands.json

# the above python script is saved in the root webkit/ dir as webkit-finalise-compiled-json.py
# will take 'raw-compile_commands.json' and write out as './compiled_commands.json'
python3 webkit-finalise-compiled-json.py -f raw-compile_commands.json -dir .

# now generate the ccls database
ccls -index=. -log-file="log.txt" -v=3
```
