---
title: "hcloud-upload-image - qcow2 Support, Smaller Snapshots"
date: 2025-05-10T15:28:01+02:00

summary: |
  New releases of hcloud-upload-image bring major upgrades: support for uploading qcow2 images and significantly smaller snapshot sizes. Plus, container image and docs site!

ShowToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
---

I released versions v1.0.0 and v1.1.0 of [**hcloud-upload-image**](https://github.com/apricote/hcloud-upload-image) in the past week, and they come with two major new features:

* ✅ Support for uploading images in **qcow2 format**
* ✅ **Smaller snapshot sizes** thanks to improved disk handling

## 🎉 qcow2 Image Support

You can now upload disk images in qcow2 format. Internally, I’m using `qemu-img dd` to convert the qcow2 to a raw image and stream it directly to the root disk.

The main catch is that qemu-img needs a seekable source file. For raw images, that wasn’t an issue because they could just be piped. But for qcow2, I now have to store the image file on the server before writing it to disk.

The rescue system only has a ramdisk with 960 MiB available by default, so that’s the maximum size for qcow2 images at the moment. The CLI and library try to validate and warn you if your image is too large.

It would be feasible to increase the size of the ramdisk or mount a volume. But most users have requested this feature for OpenSUSE MicroOS images, which comfortably fit under 960 MiB.

📌 [Pull Request #69](https://github.com/apricote/hcloud-upload-image/pull/69)
📘 [CLI Reference: Image Size](https://apricote.github.io/hcloud-upload-image/reference/cli/hcloud-upload-image_upload.html#image-size)

## 📦 Smaller Minimal Snapshot Sizes

To upload the image, hcloud-upload-image first creates a temporary server in Hetzner Cloud with `ubuntu-24.04`. While Ubuntu is never booted, the root disk still contains all the bytes from the image.

Previously, the upload would just write the image to disk, leaving behind whatever bytes weren’t overwritten. This is not a problem for the functionality because these bytes are outside the specified ranges in the partition table.

But Hetzner snapshots don’t look at partition tables — they snapshot the entire raw disk, including any leftover bytes from a previous OS installation. Because of this, the minimum (compressed) size of the uploaded images was ~0.42GiB.

[Zargony](https://github.com/zargony) suggested zeroing the disk first with `dd if=/dev/zero` first to get rid of these bytes and make the images as small as possible. That worked — but took a full minute for the default 40GiB root disk. Not great UX, and pretty annoying if your image is larger anyway, and you do not even benefit from it.

Luckily I found `blkdiscard /dev/sda` to have the same effect on the image size, but completes in ~5 seconds. I think it's reasonable to make the process 5 seconds slower for everyone to avoid the additional complexity and heuristics to decide if its worth it to zero out the disk first.

With these changes, the (compressed) size of an example Talos x86 image was reduced from 0.42GiB to 0.20GiB.

### Benefits:

1. **Smaller image size** → slightly lower Hetzner bills. For Talos, that’s about €0.0024/month (excl. VAT) — not life-changing, but hey.

2. **Faster provisioning**. When you create a new server from a custom image, the image first needs to be copied from the image storage onto the server hosting your VM. In my testing, I found that it takes roughly 2 minutes per GiB. With the smaller image, Talos servers come online ~25 seconds faster.

## Minor Features

### 📚 Docs Website

There’s now a documentation website with the CLI references and links to the official guides from the Talos, Fedora CoreOS, and Flatcar docs:

👉 [apricote.github.io/hcloud-upload-image](https://apricote.github.io/hcloud-upload-image/)

### 🐳 Container Image

There’s now a **container image** published at `ghcr.io/apricote/hcloud-upload-image`, great for local use or CI pipelines:

```bash
docker run --rm -e HCLOUD_TOKEN="<your token>" \
  ghcr.io/apricote/hcloud-upload-image:latest \
  upload <flags>
```

Thanks to [PRIHLOP](https://github.com/PRIHLOP) for suggesting the feature and implementing the first version.

This is implemented with [ko](https://ko.build). I really like how simple it is to build a container for Go with it. No need to have a Docker daemon installed; no need to wrangle `Dockerfiles`, cache mounts or multiple stages to have a small and efficient image; no need for complicated cross-compiling setups, buildx and qemu for multi-arch images.

## 🧪 Testing, Bugs & Shell Pipelines

There were also a number of bugs that came with the qcow2 support and some other smaller improvements made in v1.0.0.

That is one of the downsides of not having proper testing. As I work on this on my own time and mostly for my own purposes, testing is limited to what I need for myself.

I did end up at least [adding some unit tests](https://github.com/apricote/hcloud-upload-image/pull/98/files#diff-b0d3df197c7b8434f621a9b55218097dcaadb57ba673d32f9fad068bb79a22a4R17) for the code that builds the shell pipeline that decompresses and writes the image on the server.

Concatenating shell commands like `wget ... |`, `xz -cd | `, `dd of=/dev/sda bs=4M` is error-prone. Especially if every single one of them depends on some input flag to support:

- **file sources**: remote URL vs upload from local file
- **compression**: none, xz, bz2
- **image format**: raw, qcow2

Not sure what the best alternative to this is. I could make a shell script and use `mkfifo` to pass around the data, upload that to the server, and run it. But I have never used mkfifo before, and shell scripts frustrate me.

Or I could implement all of this in a separate Go binary that is uploaded to the server, making it easier to handle the pipes there. But I would need to figure out the build process and embedding this binary in the library.

Let's just hope that the current version is bug-free, and all users are happy with the provided feature set, and I never have to touch that part of the code again 🥲

---

Thanks to everyone who's tried out **hcloud-upload-image** so far and provided feedback.

If you have any comments or want to leave some feedback, feel free to drop them on the [Mastodon thread](https://hachyderm.io/@apricote/114483565536111800) or on the [GitHub repository](https://github.com/apricote/hcloud-upload-image/issues/new).
