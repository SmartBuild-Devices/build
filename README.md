# SmartBuild

## Table of Contents

- Why it exists
- How to build on ROMs that support it
- List of ROMs that support it
- How to build on ROMs that don't support it
- How to migrate from a "normal" device tree to a "SmartBuild" device tree
  - Modifying `AndroidProducts.mk`
  - Modifying `device.mk`
  - Creating a commonized makefile
  - Modifying the ROM makefile to point to the commonized makefile
  - Final considerations and useful suggestions
- Addendum: add SmartBuild support to your ROM
- License

## Why it exists

SmartBuild was first introduced to resolve a personal problem: I couldn't maintain two separate sets of device trees for the two ROMs I used to build back in 2019. Plus, often times I forgot to sync up fixes between the two sets, and I would waste precious time compiling with slightly outdated sets from time to time.

And then one day it hit me: I had to address this problem, and SmartBuild came to be.

The earliest iteration of SmartBuild consisted in a bash script that would modify the DT at compile time. This approach often times broke with code rebases.

The second iteration of SmartBuild consisted in a makefile fuckery that referenced variables and files and folders like ol' grandma's spaghetti at compile time. This didn't break, but it was a pain to account for in code rebases. Too many makefiles and no "simple" way to deal with the problem.

The third iteration of SmartBuild (and the first widely available to adopt) is the current one; it's based off the second iteration but with a way more clever approach, proper scalability in mind and not as many makefiles as there were before. This is the best compromise so far.

## How to build on ROMs that support it

You build on them exactly like you built before SmartBuild was even a thing.

```bash
source build/envsetup.sh
lunch <rom>_<device>-<type>
# make bacon || mka <rom> || whatever other command it is
```

## List of ROMs that support it

- RevengeOS

## How to build on ROMs that don't support it

You have to export the `SMARTBUILD_RELEASE` environment variable before sourcing `build/envsetup.sh`.

```bash
export SMARTBUILD_RELEASE=<rom>
source build/envsetup.sh
lunch <rom>_<device>-<type>
# make bacon || mka <rom> || whatever other command it is
```

## How to migrate from a "normal" device tree to a "SmartBuild" device tree

From here on out, we'll be relying on this example: porting DTs of the Xiaomi Redmi Note 5 Pro (whyred) from LineageOS (official) to SmartBuild.

### Modifying `AndroidProducts.mk`

We basically search and replace `lineage` with `$(SMARTBUILD_RELEASE)`. This is the easiest file to modify.

```diff
 PRODUCT_MAKEFILES := \
-	$(LOCAL_DIR)/lineage_whyred.mk
+	$(LOCAL_DIR)/$(SMARTBUILD_RELEASE)_whyred.mk

 COMMON_LUNCH_CHOICES := \
-	lineage_whyred-user \
-	lineage_whyred-userdebug \
-	lineage_whyred-eng
+	$(SMARTBUILD_RELEASE)_whyred-user \
+	$(SMARTBUILD_RELEASE)_whyred-userdebug \
+	$(SMARTBUILD_RELEASE)_whyred-eng
```

Keep an eye on the copyright note, you could end up with "Copyright 20XX $(SMARTBUILD_RELEASE)OS" which is hilarious to me (and I fell for it).

### Modifying `device.mk`

First off, we might want to locate the part of the makefile where properties are being read.

Look out for a line like this one:
```makefile
# Inherit properties.mk
$(call inherit-product, $(DEVICE_PATH)/properties.mk)
```

Append the following right below it:

```makefile
ifneq ($(wildcard $(DEVICE_PATH)/properties-common.mk),)
	$(call inherit-product, $(DEVICE_PATH)/properties-common.mk)
endif

ifneq ($(wildcard $(DEVICE_PATH)/properties-$(SMARTBUILD_RELEASE).mk),)
	$(call inherit-product, $(DEVICE_PATH)/properties-$(SMARTBUILD_RELEASE).mk)
endif
```

Then, we need to locate the lines which define where overlays' folders are defined. Look out for those lines:
```makefile
# Overlays
DEVICE_PACKAGE_OVERLAYS += \
	$(DEVICE_PATH)/overlay \
	$(DEVICE_PATH)/overlay-lineage
```

Here we want to nuke the `overlay-lineage` part and add some logic to load common and ROM-specific overlays right below it. Here's the complete diff:

```diff
 # Overlays
 DEVICE_PACKAGE_OVERLAYS += \
-	$(DEVICE_PATH)/overlay \
-	$(DEVICE_PATH)/overlay-lineage
+	$(DEVICE_PATH)/overlay
+
+ifneq ($(wildcard $(DEVICE_PATH)/overlay-common/.),)
+	DEVICE_PACKAGE_OVERLAYS += \
+		$(DEVICE_PATH)/overlay-common
+endif
+
+ifneq ($(wildcard $(DEVICE_PATH)/overlay-$(SMARTBUILD_RELEASE)/.),)
+	DEVICE_PACKAGE_OVERLAYS += \
+		$(DEVICE_PATH)/overlay-$(SMARTBUILD_RELEASE)
+endif
```

### Creating a commonized makefile

We need to create our commonized makefile now. To start, we copy `lineage_whyred.mk` to `common_whyred.mk`, and then, we start to trim out the lineage-specific parts of the copy.

I guess that a diff is worth more than me explaining stuff, so here it is:

```diff
 #
 # Copyright (C) 2018-2019 The LineageOS Project
 #
 # Licensed under the Apache License, Version 2.0 (the "License");
 # you may not use this file except in compliance with the License.
 # You may obtain a copy of the License at
 #
 #      http://www.apache.org/licenses/LICENSE-2.0
 #
 # Unless required by applicable law or agreed to in writing, software
 # distributed under the License is distributed on an "AS IS" BASIS,
 # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 # See the License for the specific language governing permissions and
 # limitations under the License.
 #
 
 # Inherit from those products. Most specific first.
 $(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
 $(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk)
 $(call inherit-product, $(SRC_TARGET_DIR)/product/product_launched_with_o_mr1.mk)

-# Inherit some common Lineage stuff
-$(call inherit-product, vendor/lineage/config/common_full_phone.mk)
+# Inherit from SmartBuild ROM config
+$(call inherit-product, $(SMARTBUILD_RELEASE_CONFIG))

 # Inherit from whyred device
 $(call inherit-product, $(LOCAL_PATH)/device.mk)

 PRODUCT_BRAND := Xiaomi
 PRODUCT_DEVICE := whyred
 PRODUCT_MANUFACTURER := Xiaomi
-PRODUCT_NAME := lineage_whyred
+PRODUCT_NAME ?= $(SMARTBUILD_RELEASE)_whyred
 PRODUCT_MODEL := Redmi Note 5

 PRODUCT_GMS_CLIENTID_BASE := android-xiaomi

 TARGET_VENDOR_PRODUCT_NAME := whyred

 PRODUCT_BUILD_PROP_OVERRIDES += \
     PRIVATE_BUILD_DESC="whyred-user 8.1.0 OPM1.171019.011 V9.5.11.0.OEIMIFA release-keys"

 BUILD_FINGERPRINT := xiaomi/whyred/whyred:8.1.0/OPM1.171019.011/V9.5.11.0.OEIMIFA:user/release-keys
```

### Modifying the ROM makefile to point to the commonized makefile

Now we need to modify `lineage_whyred.mk` and remove everything that is in `common_whyred.mk`. Then we'll include the common file at the end.

Also, we need to put the "common Lineage stuff" into a variable which will be picked up by the common file and properly inherited.

The final lineage makefile will look like this:

```makefile
# Inherit some common Lineage stuff
SMARTBUILD_RELEASE_CONFIG := vendor/lineage/config/common_full_phone.mk

include $(LOCAL_PATH)/common_whyred.mk
```

### Final considerations and useful suggestions

#### Keeping your trees as tidy as possible

It's best practice if
- you keep all the vanilla AOSP-compatible overlays and properties in `overlay/` and `properties.mk`;
- you keep all the commonly available custom ROM overlays and properties in `overlay-common/` and `properties-common.mk`
- you keep all the ROM-specific overlays and properties in `overlay-<rom>/` and `properties-<rom>.mk`

#### Taming LineageOS

I love LineageOS. I hate LineageOS when using SmartBuild. Let's go through this together anyway.

- LineageOS enforcing RRO overlays, while your other ROMs don't care or are badly affected by it.
  On `device.mk`, encapsulate the property:
  ```makefile
  ifeq ($(SMARTBUILD_RELEASE),lineage)
      PRODUCT_ENFORCE_RRO_TARGETS := *
  endif
  ```
- LineageOS messing up with its custom sepolicy rules
   Ideally you might want to create a `sepolicy-lineage/` folder and put all the Lineage-specific stuff there. To include that folder, then, you could put some logic in `BoardConfig.mk`.
   ```makefile
   ifneq ($(wildcard $(DEVICE_PATH)/sepolicy-$(SMARTBUILD_RELEASE)/.),)
       BOARD_VENDOR_SEPOLICY_DIRS += $(DEVICE_PATH)/sepolicy-$(SMARTBUILD_RELEASE)/vendor
       BOARD_PLAT_PUBLIC_SEPOLICY_DIR += $(DEVICE_PATH)/sepolicy-$(SMARTBUILD_RELEASE)/public
       BOARD_PLAT_PRIVATE_SEPOLICY_DIR += $(DEVICE_PATH)/sepolicy-$(SMARTBUILD_RELEASE)/private
   endif
   ```
   If you are in a spiky situation where you need to check when the ROM is NOT Lineage to inherit some other sepolicy rules, then you might also want to add this bit:
   ```makefile
   ifneq ($(SMARTBUILD_RELEASE),lineage)
       BOARD_VENDOR_SEPOLICY_DIRS += $(DEVICE_PATH)/sepolicy-not-lineage/vendor
       BOARD_PLAT_PUBLIC_SEPOLICY_DIR += $(DEVICE_PATH)/sepolicy-not-lineage/public
       BOARD_PLAT_PRIVATE_SEPOLICY_DIR += $(DEVICE_PATH)/sepolicy-not-lineage/private
   endif
   ```
   Why is `hal_attribute_lineage()` a thing? It breaks sepolicy in non-Lineage ROMs! Oh well, anyway.

#### POSP and the board/product separation

Hack around it with some makefile logic (as usual by now).

On `device.mk`:

```makefile
ifeq ($(SMARTBUILD_RELEASE),potato)
	# Board
	PRODUCT_BOARD_PLATFORM := <your board platform>
	PRODUCT_USES_QCOM_HARDWARE := true
endif
```

If you want to go the full mile, incapsulate the board property on `BoardConfig.mk` as well:

```makefile
ifneq ($(SMARTBUILD_RELEASE),potato)
    # QCOM hardware
    BOARD_USES_QCOM_HARDWARE := true
endif
```

## Addendum: add SmartBuild support to your ROM

Cherry-pick [this commit](https://github.com/RevengeOS/android_vendor_revengeos/commit/d6a60edd2be2cae1fb3c11a69cad7f35c9734f85) (modify it to match your ROM).

## License

```
#
# Copyright (C) 2020 Naomi Calabretta
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
```
