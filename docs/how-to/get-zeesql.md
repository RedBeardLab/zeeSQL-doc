# How to get zeeSQL

There are different way to get `zeeSQL` each suitable for production and testing.

The best way for you dependes on your specific use case.

## Use the docker image

The simplest way to obtain `zeeSQL` is to use the standard docker image: `redbeardlab/zeesql`.

This particular image is based on the standard `Redis` image.

It starts `Redis` and automatically loads `zeeSQL` for you.

As soon as the image start, `zeeSQL` is loaded and ready to be used.

This can be used with `docker`, `podman`, but also in `kubernetes`.

If you are using docker, make sure to expose the port `6379` for `Redis`.

**Example**

```text
docker run -d --name zeesql -p 6379:6379 --rm redbeardlab/zeesql
```

## Getting the binary

The second option is to get the `zeeSQL` binary.

The binary is rather small, ~10MB, and it can be downloaded from:

```text
https://zeesql.com/releases/latest/zeesql.so
```

A simple way to get the binary is:

```text
wget https://zeesql.com/releases/latest/zeesql.so
```

The binary can be distributed in whichever way you like, so you can push it in your private docker image, or send it to customers.

