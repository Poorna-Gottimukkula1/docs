# Reproduction Steps for Issue using CoreOS Assembler (COSA)

## Prerequisites

- Ensure `podman` is installed
- Access to `quay.io` to pull the COSA container image
- Ref DOC https://coreos.github.io/coreos-assembler/building-fcos/ for more details.
---

## Setup COSA Environment

Define the `cosa` helper function:

```bash
cosa() {
    env | grep COREOS_ASSEMBLER
    local -r COREOS_ASSEMBLER_CONTAINER_LATEST="quay.io/coreos-assembler/coreos-assembler:latest"

    if [[ -z ${COREOS_ASSEMBLER_CONTAINER} ]] && podman image exists ${COREOS_ASSEMBLER_CONTAINER_LATEST}; then
        local -r cosa_build_date_str="$(podman inspect -f "{{.Created}}" ${COREOS_ASSEMBLER_CONTAINER_LATEST} | awk '{print $1}')"
        local -r cosa_build_date="$(date -d ${cosa_build_date_str} +%s)"

        if [[ $(date +%s) -ge $((cosa_build_date + 60*60*24*7)) ]]; then
            echo -e "\e[0;33m----" >&2
            echo "The COSA container image is more than a week old and likely outdated." >&2
            echo "You should pull the latest version with:" >&2
            echo "podman pull ${COREOS_ASSEMBLER_CONTAINER_LATEST}" >&2
            echo -e "----\e[0m" >&2
            sleep 10
        fi
    fi

    set -x
    podman run --rm -ti --security-opt=label=disable --privileged \
        --userns=keep-id:uid=1000,gid=1000 \
        -v=${PWD}:/srv/ --device=/dev/kvm --device=/dev/fuse \
        --tmpfs=/tmp -v=/var/tmp:/var/tmp --name=cosa \
        ${COREOS_ASSEMBLER_CONFIG_GIT:+-v=$COREOS_ASSEMBLER_CONFIG_GIT:/srv/src/config/:ro} \
        ${COREOS_ASSEMBLER_GIT:+-v=$COREOS_ASSEMBLER_GIT/src/:/usr/lib/coreos-assembler/:ro} \
        ${COREOS_ASSEMBLER_ADD_CERTS:+-v=/etc/pki/ca-trust:/etc/pki/ca-trust:ro} \
        ${COREOS_ASSEMBLER_CONTAINER_RUNTIME_ARGS} \
        ${COREOS_ASSEMBLER_CONTAINER:-$COREOS_ASSEMBLER_CONTAINER_LATEST} "$@"
    rc=$?
    set +x
    return $rc
}
```

Pull Latest COSA Container
```bash
podman pull quay.io/coreos-assembler/coreos-assembler:latest
```


Initialize Build Directory
```bash
mkdir rawhide-test && cd rawhide-test
```
Initialize the Fedora CoreOS config:
```bash
cosa init https://github.com/coreos/fedora-coreos-config.git --branch rawhide
```

Build Images

Build the OSTree artifact:
```bash
cosa build
```
Build additional formats:
```bash
cosa osbuild qemu metal metal4k live
```

## Running Tests
## Running Reprovision Tests (Test Group)

Use the following command to run the **reprovision test group**:

```bash
cosa kola run \
  --rerun \
  --allow-rerun-success=tags=needs-internet \
  --build=latest \
  --on-warn-failure-exit-77 \
  --arch=ppc64le \
  --tag=reprovision
```
Upgrade Tests

```bash
cosa kola run-upgrade \  --rerun \  --allow-rerun-success=tags=needs-internet \  --build=latest \  --on-warn-failure-exit-77 \  --arch=ppc64le \  --upgrades
```

### Notes
- In this case, we are not providing a QCOW2 image explicitly.
- Kola automatically uses the latest locally built image from the default COSA build directory (./builds/latest/ppc64le/).
- This works because the image was already built using cosa build and cosa osbuild.
- Use this approach when testing locally built artifacts instead of pre-built images.

## Running Tests with Pre-built Images
### Run a Single Test

Use the following command to run a specific Kola test with a pre-built QCOW2 image:

```bash
cosa kola run -p qemu \
  --qemu-image <path-to-qcow2-image> \
  'ext.config.rpm-ostree.kernel-replace'
```
Example:
```bash
cosa kola run -p qemu \  --qemu-image fedora-coreos-45.20260421.91.1-qemu.ppc64le.qcow2 \  'ext.config.rpm-ostree.kernel-replace'
```

Notes
- Ensure /dev/kvm is available for virtualization support
- Tests requiring internet access use the tag needs-internet
- Replace the QCOW2 image path as needed when testing pre-built images
- You can run a specific test in debug mode to get more detailed logs: `cosa kola run --debug <test-name>`  
    - Using --debug will print additional information, including the QEMU command used to start the virtual machine.
    - This can be helpful for reproducing issues manually or inspecting how the VM is being launched.
