# How to check a downloaded GGUF against its published SHA-256, and what UCompute's development builds check

*Development build, October 2026. UCompute is in development. There is no public release or download yet. The first half of this note applies to any GGUF file from any tool.*

A GGUF file is a few hundred megabytes to tens of gigabytes of weights that you are about to hand to an inference engine. Downloads get cut off, mirrors go stale, and a file with the same name can be replaced upstream. Checking the hash takes a minute and tells you one precise thing: whether the bytes on your disk are the bytes the Hub lists for that file.

## Where Hugging Face publishes the hash

Large files on the Hugging Face Hub, including GGUF weights, are tracked as Git LFS objects, and the SHA-256 of each such file is its LFS object id; repositories on the Hub's newer Xet storage still publish it. You can read it without downloading the file, in two ways.

The raw pointer (replace the repository and file name):

```sh
curl -sL https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/raw/main/qwen2.5-0.5b-instruct-q4_k_m.gguf
```

```
version https://git-lfs.github.com/spec/v1
oid sha256:74a4da8c9fdbcd15bd1f6d01d621410d31c6fc00986f5eb687824e7b93d7a9db
size 491400032
```

Or the repository tree API, which lists every file with its size and, for files stored in LFS (all the GGUF files here), an `lfs.oid` that is the SHA-256. Ignore the top-level `oid`: it is the 40-character Git object id, not the file's hash.

```sh
curl -s https://huggingface.co/api/models/Qwen/Qwen2.5-0.5B-Instruct-GGUF/tree/main
```

Both read the `main` branch. If you downloaded from a specific commit, replace `main` with that commit id in either URL, because the file under a name can change between commits.

## Hash the file you have

Check the size first: if it differs from `size`, the download stopped early or is a different file, and there is no need to hash it.

**macOS**
```sh
shasum -a 256 qwen2.5-0.5b-instruct-q4_k_m.gguf
```

**Linux**
```sh
sha256sum qwen2.5-0.5b-instruct-q4_k_m.gguf
```

**Windows (PowerShell)**
```powershell
(Get-FileHash -Algorithm SHA256 .\qwen2.5-0.5b-instruct-q4_k_m.gguf).Hash
```

Compare the 64 hexadecimal characters with the pointer's `oid` (the part after `sha256:`) or the tree API's `lfs.oid`. Every character must match; Windows prints uppercase and the Hub lowercase, which is the same value.

## What a match does and does not prove

A match proves the bytes are identical to the file the Hub served for that name at that commit. It does not prove who trained the model, that the weights behave as the model card says, or that the file is safe to load; it only removes "corrupted or swapped in transit" from the list of possibilities. If the publisher's account or repository were changed, the pointer would change with it, which is why pinning a commit matters.

## What UCompute does (development build)

UCompute's development builds run this check themselves and pin more than the file name.

- UCompute's catalog pins each model to a SHA-256 hash and to the repository revision it is downloaded from. A download whose bytes do not match is rejected and the partial file deleted; a plain http:// URL is refused before any request is made; an interrupted download continues from where it stopped when you start it again; and a model already on disk is hashed again against its manifest each time the engine loads it, so a mismatch stops the load before the engine starts. *Demonstrated on an Apple M2 Mac (macOS 26.2) with development builds, October 2026; the platform is experimental. A matching hash proves the bytes are the ones the catalog pinned; it does not prove who trained the model or that its contents are safe.*
- For the model above, the development catalog's pinned SHA-256 is the same `74a4da8c…a9db` the Hub publishes, and its download URL names a fixed commit rather than `main`, so a later upload under the same file name would not change what is fetched; bytes that did not match the pinned hash would be rejected in any case.
- Before you download a model, UCompute shows its size, its license and whether it fits in your computer's memory. *Demonstrated on an Apple M2 Mac (macOS 26.2); the platform is experimental. Memory-fit estimates are calibrated for Apple Silicon Macs.*
- The engine is checked the same way: UCompute runs models through a pinned release of llama.cpp (MIT; b11609 in the October 2026 development builds), downloaded on first use and verified against a pinned SHA-256 hash (the digest published for that release); UCompute does not write its own inference kernels.

No platform is declared supported yet. macOS 26.2 on Apple M2 and Windows 11 on an Intel Core Ultra 7 258V with Arc 140V are experimental; Linux, other GPUs and phones (as clients) are planned.

UCompute's code is licensed under Apache-2.0. The source has not been published yet.

---

*Corrections and questions: hello@ucompute.org. Please send anything security-related there rather than posting it publicly. UCompute is in development. There is no public release or download yet.*

*Follow new notes: [Bluesky @ucompute.org](https://bsky.app/profile/ucompute.org) or the [Atom feed](https://ucompute.github.io/notes/feed.xml).*
