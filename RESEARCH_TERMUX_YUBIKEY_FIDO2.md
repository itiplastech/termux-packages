**Project Goal**

Enable YubiKey-backed FIDO2 security-key support in Termux `openssh`, using official Termux packaging and Docker workflow, with the minimum necessary Android/Termux-specific patches.

**What Has Been Successfully Patched And Proven**

- Official Termux Docker build workflow is working in this environment.
- Baseline package builds were proven earlier with `hello` and `python`.
- `libcbor` package recipe was created and built successfully.
- `libfido2` package recipe was created and built successfully.
- `openssh` recipe was updated to enable built-in security-key support with `libfido2`.
- Patched `libfido2` and patched `openssh` were successfully built and installed on Termux.
- `ssh-sk-helper` starts correctly at runtime.
- `TERMUX_USB_FD` successfully reaches the child/helper path.
- The device option is passed successfully through to `sk-usbhid.c`.
- The OpenSSH side of the pipeline is now proven far enough to show the request reaching `fido_dev_open()`.

**Current Confirmed Runtime Status**

Latest confirmed runtime log:

```text
check_enroll_options: requested device /dev/bus/usb/002/002
sk_open: fido_dev_open /dev/bus/usb/002/002 failed: FIDO_ERR_INTERNAL
```

**What This Means**

The remaining blocker is **no longer**:

- OpenSSH configure flags
- built-in security-key enablement
- `ssh-sk-helper` startup
- `FD_CLOEXEC`
- device-option passing
- helper-child propagation of `TERMUX_USB_FD`

Those parts are now effectively proven.

The likely remaining blocker is now in the **Android USB transport layer**, specifically one of:

- `libfido2`
- `hidapi`
- `libusb`
- Android/Termux USB FD handling inside that stack

**Earlier Issues That Are No Longer The Main Blocker**

These were real earlier packaging/integration issues, but they are not the latest blocker now:

- `libcbor` CMake IPO/LTO issue with:
  ```text
  clang: error: invalid linker name in argument '-fuse-ld=gold'
  ```
- `libfido2.pc` / `libcrypto.pc` pkg-config path and metadata concerns
- initial `explicit_bzero` Android fallback issue involving `bzero`

Those matter for package hygiene, but the latest runtime evidence shows the build/configure path is past them.

**Files Modified**

Confirmed important modified files in this project:

- `termux-packages/packages/libcbor/build.sh`
- `termux-packages/packages/libfido2/build.sh`
- `termux-packages/packages/libfido2/0001-openbsd-compat-explicit_bzero-use-memset-fallback.patch`
- `termux-packages/packages/openssh/build.sh`

Likely active runtime-research files now present in the worktree:

- `termux-packages/packages/libfido2/0001-termux-usb-fd.patch`
- `termux-packages/packages/openssh/0001-termux-preserve-usb-fd-for-sk-helper.patch`
- `termux-packages/hid_linux.c`
- `termux-packages/hid_linux.c.orig`
- `termux-packages/ssh-sk-client.c`
- `termux-packages/ssh-sk-client.c.orig`

**Patch Files Created**

Confirmed:

- `packages/libfido2/0001-openbsd-compat-explicit_bzero-use-memset-fallback.patch`

Runtime-focused patch work now present:

- `packages/libfido2/0001-termux-usb-fd.patch`
- `packages/openssh/0001-termux-preserve-usb-fd-for-sk-helper.patch`

**Build/Test Commands Used**

Official build commands used or prepared in this project:

```powershell
.\scripts\run-docker.ps1 ./build-package.sh -a aarch64 hello
.\scripts\run-docker.ps1 ./build-package.sh -a aarch64 python
.\scripts\run-docker.ps1 ./build-package.sh -a aarch64 libcbor
.\scripts\run-docker.ps1 ./build-package.sh -a aarch64 libfido2
.\scripts\run-docker.ps1 ./build-package.sh -a aarch64 openssh
```

Useful verification flow already proven:

- build patched `libfido2`
- build patched `openssh`
- install both on Termux
- run OpenSSH security-key enrollment/runtime path with device option
- inspect helper/runtime logs

**Current Hypothesis**

The OpenSSH integration path is now mostly validated end-to-end up to the point where `libfido2` tries to open the device.

Current best hypothesis:

- OpenSSH is passing the request correctly.
- The helper is launching correctly.
- The device path is reaching `sk-usbhid.c`.
- The failure is happening at the lower USB/FIDO transport layer on Android:
  - `fido_dev_open()`
  - `hidapi`
  - `libusb`
  - Android USB FD/device access semantics

So the project has moved from “can we enable FIDO2 in OpenSSH?” to “why does the Android USB backend still fail with `FIDO_ERR_INTERNAL` when opening the real token?”

**Recommended Next Phase**

1. Focus only on runtime USB transport debugging.
2. Inspect the patched `libfido2` USB/FD path in detail.
3. Trace how `hidapi` and/or `libusb` handle the Android-passed USB FD.
4. Confirm whether `fido_dev_open()` is still trying to use normal Linux path semantics instead of the Termux Android FD path.
5. Add targeted runtime logging around:
   - `sk-usbhid.c`
   - `hid_linux.c`
   - any Termux-specific USB FD shim
6. Verify whether the token can be reached through the patched transport without relying on `/dev/bus/usb/...` normal Linux behavior.

**Things NOT To Redo Next Time**

- Do not re-check OpenSSH configure flags from scratch.
- Do not re-check whether `--with-security-key-builtin` is the right flag.
- Do not re-debug `ssh-sk-helper` startup.
- Do not re-debug device option propagation.
- Do not re-debug `TERMUX_USB_FD` reaching the helper.
- Do not treat `libfido2.pc` / `libcrypto.pc` as the current primary blocker.
- Do not restart from packaging theory; the current problem is runtime USB transport.

**Practical Resume Point**

Resume from this exact confirmed state:

```text
check_enroll_options: requested device /dev/bus/usb/002/002
sk_open: fido_dev_open /dev/bus/usb/002/002 failed: FIDO_ERR_INTERNAL
```

That is the strongest current checkpoint. The next work should stay tightly focused on why `fido_dev_open()` fails on Android/Termux after all higher-level OpenSSH plumbing has already been proven.