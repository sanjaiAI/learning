# Module 09: Android Emulator Pipelines
# மாடுல் 09: Android Emulator (Goldfish) Pipelines

---

## 🎯 What? | என்ன?

**English:** Provision Android emulator (Goldfish/Cuttlefish) infrastructure for running CTS/VTS/STS test suites in CI — AVD creation, GPU acceleration, and parallel execution.

**தமிழ்:** Android emulator infrastructure provision — CTS/VTS/STS tests CI-ல் run செய்ய AVD creation, GPU acceleration, parallel execution.

---

## 🛠️ Emulator Host Setup

```yaml
# roles/android_emulator/tasks/main.yml
---
- name: Install KVM and virtualization tools
  apt:
    name:
      - qemu-kvm
      - libvirt-daemon-system
      - bridge-utils
      - cpu-checker
    state: present

- name: Verify KVM support
  command: kvm-ok
  register: kvm_check
  changed_when: false
  failed_when: "'KVM acceleration can be used' not in kvm_check.stdout"

- name: Add CI user to kvm group
  user:
    name: "{{ ci_user }}"
    groups: [kvm, libvirt]
    append: true

- name: Install Android SDK command-line tools
  unarchive:
    src: "https://dl.google.com/android/repository/commandlinetools-linux-{{ sdk_tools_version }}_latest.zip"
    dest: "{{ android_sdk_root }}/cmdline-tools/latest"
    remote_src: true
    creates: "{{ android_sdk_root }}/cmdline-tools/latest/bin/sdkmanager"

- name: Accept SDK licenses
  command: "yes | {{ android_sdk_root }}/cmdline-tools/latest/bin/sdkmanager --licenses"
  changed_when: false

- name: Install SDK packages
  command: >
    {{ android_sdk_root }}/cmdline-tools/latest/bin/sdkmanager
    "platform-tools"
    "platforms;android-{{ android_api_level }}"
    "system-images;android-{{ android_api_level }};google_apis;x86_64"
    "emulator"
  environment:
    ANDROID_SDK_ROOT: "{{ android_sdk_root }}"
  changed_when: false

- name: Create AVD
  command: >
    {{ android_sdk_root }}/cmdline-tools/latest/bin/avdmanager create avd
    --name "{{ avd_name }}"
    --package "system-images;android-{{ android_api_level }};google_apis;x86_64"
    --device "pixel_6"
    --force
  become_user: "{{ ci_user }}"
  environment:
    ANDROID_SDK_ROOT: "{{ android_sdk_root }}"

- name: Configure AVD for CI (headless, performance)
  copy:
    content: |
      hw.gpu.enabled=yes
      hw.gpu.mode=swiftshader_indirect
      hw.ramSize=4096
      hw.cpu.ncore=4
      disk.dataPartition.size=8G
      hw.keyboard=yes
      hw.lcd.density=440
    dest: "/home/{{ ci_user }}/.android/avd/{{ avd_name }}.avd/config.ini"
    owner: "{{ ci_user }}"
```

---

## 🛠️ Emulator Launch & Test Scripts

```yaml
- name: Deploy emulator launch script
  template:
    src: start-emulator.sh.j2
    dest: /opt/android-ci/start-emulator.sh
    mode: '0755'

- name: Deploy CTS runner script
  template:
    src: run-cts.sh.j2
    dest: /opt/android-ci/run-cts.sh
    mode: '0755'
```

```bash
# templates/start-emulator.sh.j2
#!/bin/bash
# Start emulator headless for CI
export ANDROID_SDK_ROOT={{ android_sdk_root }}

$ANDROID_SDK_ROOT/emulator/emulator \
  -avd {{ avd_name }} \
  -no-window \
  -no-audio \
  -gpu swiftshader_indirect \
  -no-snapshot \
  -wipe-data \
  -memory 4096 \
  -cores 4 &

# Wait for boot
echo "Waiting for emulator boot..."
adb wait-for-device
timeout 300 adb shell 'while [[ -z $(getprop sys.boot_completed) ]]; do sleep 5; done'
echo "Emulator booted successfully!"
```

```bash
# templates/run-cts.sh.j2
#!/bin/bash
# Run CTS/VTS suite
SUITE=$1    # cts, vts, sts
MODULE=$2   # Optional: specific module

cd /opt/android-cts/{{ cts_version }}/android-cts/tools

if [ -n "$MODULE" ]; then
    ./cts-tradefed run commandAndExit cts --module "$MODULE" --skip-device-info
else
    ./cts-tradefed run commandAndExit cts --skip-device-info
fi
```

---

## 🛠️ Parallel Emulator Execution

```yaml
# Run multiple emulators on same host (different ports)
- name: Create multiple AVDs for parallel testing
  command: >
    {{ android_sdk_root }}/cmdline-tools/latest/bin/avdmanager create avd
    --name "avd-{{ item }}"
    --package "system-images;android-{{ android_api_level }};google_apis;x86_64"
    --force
  loop: "{{ range(1, parallel_emulators + 1) | list }}"
  become_user: "{{ ci_user }}"

- name: Deploy parallel launcher
  template:
    src: parallel-emulators.sh.j2
    dest: /opt/android-ci/parallel-emulators.sh
    mode: '0755'
```

---

## 📋 Cheat Sheet | விரைவு குறிப்பு

```
┌──────────────────────────────────────────────────┐
│    ANDROID EMULATOR CI CHEAT SHEET               │
├──────────────────────────────────────────────────┤
│ HOST REQUIREMENTS:                               │
│   KVM enabled (Intel VT-x / AMD-V)              │
│   16+ GB RAM per emulator instance              │
│   SSD (emulator I/O intensive)                  │
│   kvm group membership for CI user              │
│                                                  │
│ EMULATOR FLAGS (CI):                             │
│   -no-window (headless)                          │
│   -no-audio                                      │
│   -gpu swiftshader_indirect (software GPU)       │
│   -no-snapshot (clean state each run)            │
│   -wipe-data (fresh data partition)              │
│                                                  │
│ TEST SUITES:                                     │
│   CTS = Compatibility Test Suite (Google req)    │
│   VTS = Vendor Test Suite (HAL tests)           │
│   STS = Security Test Suite                      │
│                                                  │
│ PARALLEL STRATEGY:                               │
│   Multiple AVDs on same host (diff ports)        │
│   Or: K8s pods with KVM device passthrough       │
│   Sharding: split CTS modules across instances   │
└──────────────────────────────────────────────────┘
```

---

## ✅ Self-Check | சுய மதிப்பீடு

- [ ] Android emulator host provision முடியும்
- [ ] AVD create and configure முடியும்
- [ ] Headless emulator for CI launch முடியும்
- [ ] CTS/VTS execution automate முடியும்
