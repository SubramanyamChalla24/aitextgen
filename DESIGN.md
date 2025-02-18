# Design

A few notes on some opinionated design decisions present in aitextgen.

## Features

- TokenDataset caching is done via [MessagePack](https://msgpack.org/index.html). Tokens are encoded as integers, which is what MessagePack is especially designed for, compressing the final file by about 50%. MessagePack will also save/reload the file much faster than `pickle` or `json`.

  - By default, this is additionally compressed via `gzip`, further reducing file size. This may have memory constraints for super-large files (1GB+), hence a parameter to disable compression.

- Although GPT-2 is the default Transformers model architecture used, there is limited GPT-2 specific code. This allows this tool to easily adapt to new CLM architectures that may be released, such as SparseTransformers or Reformers.

- `generate_to_file()` automatically assigns the generated process a seed if one is not specified. This allows other users to reproduce a generation deterministically (e.g. in a Jupyter Notebook), in order to provide proof that a text was generated by AI and not altered.

- For training checkpoints, aitextgen deliberately disables pytorch-Lightning’s checkpoint feature since it incurs a lot of overhead, and using the native model saving within the model itself is easier.

- Testing/Validation is deliberately not implemented since it serves as more of a crutch and doesn’t provide that much help in identifying over-fitting.

## Philosophies

- The development intent of aitextgen is as a _tool_ for AI text generation, and not a philosophical experiment behind AI consciousness or whatnot. (alternatively, one could argue that _humans_ are the ones who perform actions based on prior knowledge and [free will is a myth](https://www.youtube.com/watch?v=kQjb-EP2JEE), but that’s a discussion for another time)

- AI generated text should _never_ be called “deep fake text.” Stop trying to make deep fake text happen.

## Deviations from Huggingface Transformers

- While aitextgen was in development, Huggingface added a fine-tuning `Trainer` to Transformers, which would make this package somewhat redundant. However, pytorch-lightning has several specific features and optimizations that make separating out the logic more efficient and easier to modify/tweak.

- For generation, random sampling is the default while in the base Transformers, [greedy sampling is used](https://github.com/huggingface/transformers/pull/3298) which makes the output deterministic.

- The TokenDataset uses an implementation different from Transformer’s TextDataset, as that implementation is flawed for GPT-2. The Transformers TextDataset splits the corpus into blocks of `block_size` which means a) many token subsets may not be possible to be fed into the model and b) Merges may be split across the block size boundaries, causing incorrect tokenization. TokenData fixes both issues by creating a giant single text and provides the models random subsets of that model of `block_size`.

## Why aitextgen was made vs. improving gpt-2-simple

gpt-2-simple was a quick project intended to use GPT-2 in an app when no user-friendly alternative existed and for others to use GPT-2 without fiddling around with the command line. However, since development on GPT-2 from OpenAI abruptly and unexpectedly stopped, it would take too much time to manually implement the necessary improvements on my end.

It is faster to build a new project on a stronger, more futureproof base than it is to iterate on gpt-2-simple. Both transformers and pytorch-lightning are in rapid development, so I am confident they will be around for a while.

I am also trying to avoid feature creep, as that happened with textgenrnn and gpt-2-simple and made the packages more difficult to maintain. Not all Issues/PRs will be added.

## Deviations from gpt-2-simple

- PyTorch is the primary ML framework since it’s the first-class citizen for Transformers. Additionally, as of 2020, the differences (in both performance and usability) between TensorFlow and PyTorch have converged, so I figured I’d try PyTorch. There were also a few random issues when developing gpt-2-simple (memory leaks, having to start a new session) which I believe were attributable to the TF base; maybe PyTorch will work out better.

- The base `generate()` function in Transformers which aitextgen uses is “more correct” than gpt-2-simple: it stops and starts at special tokens negating the need for encoding hacks, it uses caching to speed up decoding, and it only generates as much text as needed.

- The OOP approach (which I had used long ago in textgenrnn) should make working with aitextgen much easier for integration into other apps.

- For generation, you need to explicitly load/move the model to the GPU via `to_gpu()` whereas TensorFlow did it automatically (it’s unnecessary to do so for training)

## Why create a separate package instead of commiting directly to Transformers

aitextgen has specific optimizations that may be out of scope for the Transformers package primary use cases. If an optimization benefits the default use cases of Transformers without negatively impacting it, I may do a pull request.

As the license of this repo is permissible (MIT), the Huggingface team is able to apply any modifications/ideas to their own repo if necessary without explicit permission.
