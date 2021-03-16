# SmartBuild version 4 - Draft

## Architecture

### Problem

We need a really flexible architecture that lets us isolate specific code paths and files if the need ever presents to us (example: Lineage based ROMs versus vanilla AOSP based ROMs).

This architecture should be achievable with as little effort as possible, if possible.

### Proposed solution

Refer to the following diagram

![Architecture diagram](https://i.imgur.com/yoSMiyO.png)

Every device tree, be it specific or common, shall be overlayed by "support layers", stackable by definition.

This way we can isolate code paths and modules with fine grained control, given to us by Makefiles.

Please consider the following example "layer routes" to build a series of common ROMs for a target device:

![Paths](https://i.imgur.com/SLQ821h.png)

In the real world, the directory tree should resemble something like this:

```   │ ├ ── └
<project root>
├── <... omitted ...>
└── device/
    ├── smartbuild/
    │   └── <common utilities for support layers>
    └── your_brand/
         └── your_device/

             ├── smartbuild/
             │   ├── aosp-common/
             │   │   ├── overlays/
             │   │   ├── some-common-package/
             │   │   │   └── Android.mk
             │   │   ├── Android.mk
             │   │   ├── BoardConfig.mk
             │   │   ├── device.mk
             │   │   └── properties.mk
             │   └── revengeos/
             │       ├── overlays/
             │       ├── some-rom-specific-package/
             │       │   └── Android.mk
             │       ├── Android.mk
             │       ├── AndroidProducts.mk
             │       ├── BoardConfig.mk
             │       ├── device.mk
             │       ├── properties.mk
             │       └── revengeos_your_device.mk
             ├── Android.mk
             └── AndroidProducts.mk
```

Keep in mind that the concept of layer stacks: each support layer is stackable, and the full stack is made by the base device tree plus all the support layers and the target layer at the end closer to the entrypoint (as in `rom_device.mk`).

## Challenges

### Defining an accurate combo and entrypoint for `lunch` to use

SmartBuild allows multiple projects to be built with the same device trees. This means that we do not have a clear entry point for each project, and that we are forced to determine it dynamically.

This is currently done via the `SMARTBUILD_RELEASE` environment variable.

The best case scenario would be to let ROMs define this variable in the vendor `envsetup.sh`, however we cannot count on projects supporting SmartBuild officially, so we need a "plan B".

Proposed by @AgentFabulous, there is this handy dandy shell script that does the job.

```shell=/bin/sh
prefix=$(declare -f breakfast | grep lunch | tail -n 1 | xargs | cut -d \  -f 2 | cut -d _ -f 1)
if [ -z $prefix ]; then
  prefix="aosp"
fi
echo $prefix
```

Problem is, how do we execute it before our device's `AndroidProducts.mk` is called?
The first idea would be to have a makefile macro to call, something like `$(call smartbuild-determine-release)`. And then, can we expose this macro to our `AndroidProducts.mk`?

There is also another problem: are we totally sure that things are not going to break that much if we have the entrypoint in the support layer, compared to having it as a glue in a common location such as the device tree folder?

### `Android.mk` is a problem

To be precise, the one at `device/your_brand/your_device` is, and it could cause duplicate/broken build rules if defined "the common way".

Thing is, we have the `smartbuild/` folder containing our support layers, but out of all of those, only the required ones shall be "active", and therefore if our beloved makefile calls all the other `Android.mk` files under it, it will also be including those that are not needed in some builds.

So, the correct way to handle `Android.mk` would be, in steps:
- Include all the makefiles in the base device tree, except those in the `smartbuild/` folder
- Include `Android.mk` for each support layer involved, in order, from the one closer to the base device tree (common) to the one closer to the target support layer, in the layer stack.
- Include `Android.mk` of the target support layer as the last one.

### `BoardConfig.mk` is a problem

We need a glue that finds and then calls the `BoardConfig.mk` closer to the target support layer. At the same time, the glue shall not be included in the base board configuration file, as this one shall be included by the ones in the support layers, in relation to the layer stack.

### File merging

In cases where file merging is needed, and those files are not merged by the build system at compile time (eg. overlays, manifests) we might have a hard time merging those programmatically.

It would be best to just override the files with user-merged ones, using makefiles.

### Custom sepolicy macros

In cases where devices rely on custom sepolicy macros, such as `hal_attribute_lineage`, we should provide a boilerplate implementation of those to allow the build to compile and then behave as expected at run time.

We could serve sepolicy boilerplate macros at `device/smartbuild/sepolicy`.

Note that Android's sepolicy is expanded using the m4 macroprocessor, so we can check for already defined macros before defining our own boilerplates.

Example:

```
#####################################
# hal_attribute_lineage(hal_name)
# Originally from LineageOS
ifdef(`hal_attribute_lineage',,`
define(`hal_attribute_lineage', `
attribute hal_$1;
expandattribute hal_$1 true;
attribute hal_$1_client;
expandattribute hal_$1_client true;
attribute hal_$1_server;
expandattribute hal_$1_server false;
')
')
```

## Proof of concept

There are no PoC device trees yet, since it is unclear how to solve major problems with makefiles. Once those are met with a solution, then this section will be updated.