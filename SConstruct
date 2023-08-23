#!/usr/bin/env python

'''
This file is partially derived from [godotengine/godot-cpp](https://github.com/godotengine/godot-cpp)'s `SConstruct` file. See [tools/platform/license.md](tools/platform/license.md) for license.
'''

import sys
import os
import subprocess
import pathlib

from SCons.Errors import UserError

from tools import build_utils


EnsureSConsVersion(4, 0)


try:
	Import("env")
except:
	# Default tools with no platform defaults to gnu toolchain.
	# We apply platform specific toolchains via our custom tools.
	env = Environment(tools=["default"], PLATFORM="")


# Default num_jobs to local cpu count if not user specified.
build_utils.set_default_num_jobs(env)


# to enable color output during builds
if 'TERM' in os.environ:
	env['ENV']['TERM'] = os.environ['TERM']

# hide intermediate build artifacts
env["OBJPREFIX"] = '.'
env["SHOBJPREFIX"] = '.'


# Custom options and profile flags.
opts = Variables(build_utils.get_custom_paths(), ARGUMENTS)

opts.Add(
	EnumVariable(
		"platform",
		"Target platform",
		default=env.get("platform", build_utils.detect_platform()),
		allowed_values=build_utils.platforms,
		ignorecase=2,
	)
)

# Editor and template_debug are compatible (i.e. you can use the same binary for Godot editor builds and Godot debug templates).
# Godot release templates are only compatible with "template_release" builds.
# For this reason, we default to template_debug builds, unlike Godot which defaults to editor builds.
opts.Add(
	EnumVariable(
		key="target",
		help="Compilation target",
		default=env.get("target", "template_debug"),
		allowed_values=("editor", "template_release", "template_debug"),
	)
)

opts.Add(
	EnumVariable(
		key="precision",
		help="Set the floating-point precision level",
		default=env.get("precision", "single"),
		allowed_values=("single", "double"),
	)
)


# Add platform options
tools = {}
for pl in build_utils.platforms:
	tool = Tool(pl, toolpath=["tools/platform"])
	if hasattr(tool, "options"):
		tool.options(opts)
	tools[pl] = tool


# CPU architecture options.
opts.Add(
	EnumVariable(
		key="arch",
		help="CPU architecture",
		default=env.get("arch", ""),
		allowed_values=build_utils.architecture_array,
		map=build_utils.architecture_aliases,
	)
)


# godot python options
opts.Add(
	PathVariable(
		key="godot",
		help="Path to the `godot` binary, used to extract the `gdextension_interface.h` and `extension_api.json` files from the engine.",
		default='',
		validator=build_utils.validate_executable,
	)
)

opts.Add(
	BoolVariable(
		key="skip_module_embed",
		help="Skip embedding the Godot module into the compiled library for faster iteration during development. Use with the GODOT_PYTHON_MODULE_LIB_DIR environment variable.",
		default=False,
	)
)


# Targets flags tool (optimizations, debug symbols)
target_tool = Tool("targets", toolpath=["tools/platform"])
target_tool.options(opts)


opts.Update(env)
Help(opts.GenerateHelpText(env))


# Process CPU architecture argument.
build_utils.process_arch(env)


tool = Tool(env["platform"], toolpath=["tools/platform"])

if tool is None or not tool.exists(env):
	raise ValueError("Required toolchain not found for platform " + env["platform"])

tool.generate(env)
target_tool.generate(env)


# Detect and print a warning listing unknown SCons variables to ease troubleshooting.
unknown = opts.UnknownVariables()
if unknown:
	print("WARNING: Unknown SCons variables were passed and will be ignored:")
	for item in unknown.items():
		print("	" + item[0] + "=" + item[1])


print("Building for architecture " + env["arch"] + " on platform " + env["platform"])


# Require C++20
if env.get("is_msvc", False):
	env.Append(CXXFLAGS=["/std:c++20"])
else:
	env.Append(CXXFLAGS=["-std=c++20"])

if env["precision"] == "double":
	env.Append(CPPDEFINES=["REAL_T_IS_DOUBLE"])


scons_cache_path = os.environ.get("SCONS_CACHE")
if scons_cache_path is not None:
	CacheDir(scons_cache_path)
	Decider("MD5")


# ensure generated directory exists
generated_path = pathlib.Path('src/.generated')
generated_path.mkdir(exist_ok=True)

# write mtime of godot binary, this is faster then using the binary as a dependency directly
_godot_mtime = (generated_path / '.godot-mtime')
_godot_mtime.write_text(str(pathlib.Path(build_utils.get_executable_path('godot', env)).stat().st_mtime) + '\n')


# gather sources

sources = []
python_sources = []

sources.extend(pathlib.Path('src').glob('**/*.cpp'))

python_sources.extend(pathlib.Path('lib').glob('**/*.py'))
python_sources.extend(pathlib.Path('lib').glob('**/*.pyc'))
python_sources.append('lib/godot/_internal/extension_api.json')

# filter
sources = [os.fspath(path) for path in sources if '.generated' not in path.parts]
python_sources = [os.fspath(path) for path in python_sources]

# add back after filtering
sources.append(os.fspath(generated_path / 'godot_module_archive.cpp'))

if env.get('single_source', False):
	single_source_path = (generated_path / 'single_source.cpp')
	single_source_path.write_text(
		'\n'.join(f'#include "{source.removeprefix("src/")}"' for source in sources) + '\n')

	sources[:] = [os.fspath(single_source_path)]


# init builders
from tools import builders
builders.init(env)


# setup targets

env.Alias("extract_api", [
	env.ExtractAPI(
		target = [
			'extern/gdextension/gdextension_interface.h',
			'extern/gdextension/extension_api.json',
			'lib/godot/_internal/extension_api.json',
			os.fspath(generated_path / 'gdextension_interface.h'),
		],
		source = os.fspath(_godot_mtime)
	)
])

env.Alias("generate_gdextension_api_table", [
	env.GenerateGDExtensionAPITable(
		target = os.fspath(generated_path / 'gdextension_api_table.h'),
		source = 'extern/gdextension/gdextension_interface.h',
	)
])

env.Alias("archive_importer_r_string", [
	env.MakeRString(
		target = os.fspath(generated_path / 'archive_importer_r_string.h'),
		source = 'lib/godot/_internal/utils/archive_importer.py',
	)
])

if not env.get('skip_module_embed', False):
	# pkg_files = Install('src', files)
	godot_zip = Zip(
			target = os.fspath(generated_path / 'godot.zip'),
			source = python_sources,
			ZIPROOT = 'lib',
		)
	Alias('godot_zip', godot_zip)
else:
	godot_zip = None

env.Alias("godot_module_archive", [
	env.MakeGodotModuleArchive(
		target = os.fspath(generated_path / 'godot_module_archive.cpp'),
		source = godot_zip,
	)
])


# TODO: build python
# TODO: freeze standard library


# set flags

with_lto = env.get('with_lto', False)
strip = env.get('strip', False)

env.Append(CCFLAGS = ['-fvisibility=hidden', *['-flto'] * with_lto]) # XXX
env.Append(LINKFLAGS = ['-fvisibility=hidden', *['-flto'] * with_lto, *['-s'] * strip]) # XXX

def _run_pyconfig(*args) -> list[str]:
	pyconfig_path = 'python-config'
	return subprocess.run([pyconfig_path, *args], capture_output=True, text=True).stdout.split()

env.Append(LINKFLAGS = _run_pyconfig('--ldflags'))
env.Append(LIBS = [lib.removeprefix('-l') for lib in _run_pyconfig('--embed', '--libs')])


# XXX: is this the best way to get the library path?
import sysconfig
env.Append(CPPDEFINES = [f'PYTHON_LIBRARY_PATH=\\"{sysconfig.get_config_var("LDLIBRARY")}\\"'])


# set include paths
env.Append(CPPPATH = _run_pyconfig('--includes'))
env.Prepend(CPPPATH=['src', os.fspath(generated_path), 'extern/pybind11/include'])


# library name
suffix = ".{}.{}".format(env["platform"], env["target"])
if env.dev_build:
	suffix += ".dev"
if env["precision"] == "double":
	suffix += ".double"
suffix += "." + env["arch"]
if env["ios_simulator"]:
	suffix += ".simulator"

env["suffix"] = suffix

env["OBJSUFFIX"] = suffix + env["OBJSUFFIX"]

library_name = "libgodot-python{}{}".format(env["suffix"], env["SHLIBSUFFIX"])

library = env.SharedLibrary(
	target = f"bin/{library_name}",
	source = sources,
)

Default(library)
Return("env")

