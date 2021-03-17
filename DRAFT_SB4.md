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
    │   └── <OPTIONAL common utilities for support layers>
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
             │       └── smartbuild.mk
             ├── Android.mk
             ├── AndroidProducts.mk
             ├── revengeos_your_device.mk -> smartbuild_your_device.mk
             └── smartbuild_your_device.mk
```

Keep in mind that the concept of layer stacks: each support layer is stackable, and the full stack is made by the base device tree plus all the support layers and the target layer at the end closer to the entrypoint (as in `rom_device.mk`).

## Common support layers

Those are the commonized support layers that include rules for a subset of custom ROMs.

They should be prefixed with `-common` to be easily recognizable.

Examples: `aosp-common`, `lineage-common`.

They can be inserted in an inheritance chain by any target support layer.

**THEY MUST NOT INCLUDE `smartbuild.mk`**

## Target support layers

Those are the custom ROM-specific support layers.

They are commonly named after the ROM's name in the lunch combo. Example: `aosp_<device>` gives us the `aosp` target layer. However, they can be named arbitrarily.

**THEY MUST INCLUDE `smartbuild.mk`**

## The `SMARTBUILD_LUNCH_OPT` variable

This variable is important as it is used when defining lunch combos.

If it is not set ahead of time by either the maintainer or the ROM developers, it must be defaulted to the value held in `SMARTBUILD_RELEASE` before passing it to `PRODUCT_MAKEFILES` and `COMMON_LUNCH_CHOICES` in the `AndroidProducts.mk` file.

This allows to make distinction between two or more custom ROMs that use the same lunch combo name to build. It is remarkably common to see vanilla AOSP based ROMs making use of the `aosp` name for their lunch configurations.

## The `smartbuild.mk` file

This file declares us all the ROM specific variables, like ROM build type (official or community), vendor configuration, inclusion of Google Apps (where supported); but most importantly, it defines the inheritance stack.

An example of such file is as follows:
```mk
# Inherit some common AOSP stuff
SMARTBUILD_RELEASE_CONFIG := vendor/aosp/common.mk

# Define layer inherit stack
# More generic first, more specific last
SMARTBUILD_INHERIT_STACK := \
    our-nocutoutoverlay \
    aosp-common
```

In this case, we have two support layers before our target layer, namely `our-nocutoutoverlay` and `aosp-common`; do keep in mind that the order of the stack is important. Generic layers first, specific layers last.

## The entry point: `smartbuild_<device>.mk`

This is the general equivalent to `<rom>_<device>.mk`.

It is similar to what we had in SmartBuild v3.

An example:

```mk
# Include SmartBuild properties for current release.
include $(LOCAL_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/smartbuild.mk

# Inherit from those products. Most specific first.
$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/product_launched_with_o_mr1.mk)

# Inherit from ROM
$(call inherit-product, $(SMARTBUILD_RELEASE_CONFIG))

# Inherit from device
$(call inherit-product, $(LOCAL_PATH)/device.mk)

PRODUCT_BRAND := Some brand
PRODUCT_DEVICE := device
PRODUCT_MANUFACTURER := Some manufacturer
PRODUCT_NAME := $(SMARTBUILD_LUNCH_OPT)_device
PRODUCT_MODEL := Some model
< ... other content is omitted ... >
```

Notice the usage of `SMARTBUILD_RELEASE`, `SMARTBUILD_LUNCH_OPT` and `SMARTBUILD_RELEASE_CONFIG`; those variables dynamically shape the entry point to serve our purposes well.

## Glues

### `rom_device.mk`

Do the following in a console, for each new ROM you support:
```shell=/bin/sh
$ ln -s smartbuild_device.mk rom_device.mk
```
This is currently unavoidable. Moving the entry point inside the `smartbuild/` folder makes soong go crazy; defining `smartbuild_device.mk` as the only entry makefile gives out an error because Android's build system expects lunch combos and makefiles to go coupled.

### `AndroidProducts.mk`

```mk
PRODUCT_MAKEFILES := \
    $(LOCAL_DIR)/$(SMARTBUILD_LUNCH_OPT)_device.mk

COMMON_LUNCH_CHOICES := \
    $(SMARTBUILD_LUNCH_OPT)_device-user \
    $(SMARTBUILD_LUNCH_OPT)_device-userdebug \
    $(SMARTBUILD_LUNCH_OPT)_device-eng
```

### `Android.mk`

This is an example and most likely it will not fit every device tree.

You are welcome to take inspiration from this but do your own modifications when implementing this logic to your trees.

```mk
LOCAL_PATH := $(call my-dir)

ifeq ($(TARGET_DEVICE),device)
  # Include all makefiles NOT in smartbuild/
  temp_find_leaves_excludes=$(FIND_LEAVES_EXCLUDES)
  FIND_LEAVES_EXCLUDES := $(addprefix --prune=, smartbuild)

  subdir_makefiles=$(call first-makefiles-under,$(LOCAL_PATH))
  $(foreach mk,$(subdir_makefiles),$(info including $(mk) ...)$(eval include $(mk)))

  FIND_LEAVES_EXCLUDES := $(temp_find_leaves_excludes)

  # Traverse SmartBuild inheritance tree to inherit the "active" makefiles
  $(foreach layer,$(SMARTBUILD_INHERIT_STACK), \
    $(if $(wildcard $(LOCAL_PATH)/smartbuild/$(layer)/Android.mk), \
      $(info including $(LOCAL_PATH)/smartbuild/$(layer)/Android.mk ...) \
      $(eval include $(LOCAL_PATH)/smartbuild/$(layer)/Android.mk) \
    ,) \
  )

  # Add the ROM specific Android.mk to the roster
  $(if $(wildcard $(LOCAL_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/Android.mk), \
    $(info including $(LOCAL_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/Android.mk ...) \
    $(eval include $(LOCAL_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/Android.mk) \
  ,)
endif
```

### `device.mk`

Add this at the end of the file, right before importing `vendor`.

```mk
# Inherit from SmartBuild layer stack
$(foreach layer, $(SMARTBUILD_INHERIT_STACK), \
    $(call inherit-product-if-exists, $(DEVICE_PATH)/smartbuild/$(layer)/device.mk) \
)
$(call inherit-product-if-exists, $(DEVICE_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/device.mk)
```

### `BoardConfig.mk`

Add this at the end of the file, right before importing `vendor`.

```mk
# Inherit from SmartBuild layer stack
$(foreach layer,$(SMARTBUILD_INHERIT_STACK), \
  $(if $(wildcard $(DEVICE_PATH)/smartbuild/$(layer)/BoardConfig.mk), \
    $(eval include $(DEVICE_PATH)/smartbuild/$(layer)/BoardConfig.mk) \
  ,) \
)
$(if $(wildcard $(DEVICE_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/BoardConfig.mk), \
  $(eval include $(DEVICE_PATH)/smartbuild/$(SMARTBUILD_RELEASE)/BoardConfig.mk) \
,)
```

### Other files, common trees

Please refer to the SmartBuild-Devices GitHub organization for more examples and usage within common trees.

### Interoperability with SmartBuild v3

Both versions (v3 and v4) rely on `SMARTBUILD_RELEASE` and they are interoperable to some degree.

However, it is heavily recommended to migrate to the new SmartBuild version.

### Custom sepolicy macros

In cases where devices rely on custom sepolicy macros, such as `hal_attribute_lineage`, we provide a boilerplate implementation of those to allow the build to compile and then behave as expected at run time.

To use the boilerplates, you should add `smartbuild-sepolicy-boilerplates` at the top of your `SMARTBUILD_INHERIT_STACK`. You should also `include device/smartbuild/BoardConfigSupport.mk` in your `BoardConfig.mk`

## Defining an accurate combo and entrypoint for `lunch` to use

SmartBuild allows multiple projects to be built with the same device trees. This means that we do not have a clear `lunch` combo definition for each project, and that we are forced to determine it dynamically.

The best case scenario would be to let ROMs define the `SMARTBUILD_LUNCH_OPT` variable in their vendor `envsetup.sh`, however we cannot count on projects supporting SmartBuild officially, so we need a fallback plan.

Proposed by @AgentFabulous, there is this handy dandy shell script that does the job.

```shell=/bin/sh
prefix=$(declare -f breakfast | grep lunch | tail -n 1 | xargs | cut -d \  -f 2 | cut -d _ -f 1)
if [ -z $prefix ]; then
  prefix="aosp"
fi
echo $prefix
```

This "heuristic" way of determining the right `lunch` combo has been merged into the support package.

You can use it by including `device/smartbuild/make/product.mk` and calling `$(call smartbuild-determine-lunch-opt)` anywhere before invoking `PRODUCT_MAKEFILES` and `COMMON_LUNCH_CHOICES`.

# Unsolved challenges

## File merging

In cases where file merging is needed, and those files are not merged by the build system at compile time (eg. overlays, manifests) we might have a hard time merging those programmatically.

It would be best to just override the files with user-merged ones, using makefiles.

# Proof of concept

Please check the organization for proof of concept device and common trees.