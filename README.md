# AWS CLI s3 sync for Node.js

![test](https://github.com/jeanbmar/s3-sync-client/actions/workflows/test.yml/badge.svg)

**AWS CLI ``s3 sync`` for Node.js** provides a modern client to perform S3 sync operations between file systems and S3 buckets in the spirit of the official [AWS CLI command](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/sync.html).    
AWS CLI installation is **NOT** required by this module.

## Features

- Sync a local file system with a remote Amazon S3 bucket
- Sync a remote Amazon S3 bucket with a local file system (multipart uploads are supported)
- Sync two remote Amazon S3 buckets
- Sync only new and updated objects
- Support AWS CLI options `--delete`, `--dryrun`, `--size-only`, `--include` and `--exclude`
- Support AWS SDK command input options
- Monitor object sync progress
- Sync **any** number of objects (no 1000 objects limit)
- Transfer objects concurrently
- Manage differences in folder structures easily through relocation

## Why should I use this module?

1. There is no way to achieve S3 sync using the AWS SDK for JavaScript v3 alone.
1. AWS CLI installation is **NOT** required.
1. The package contains no external dependency.
1. The AWS SDK peer dependency is up-to-date ([AWS SDK for JavaScript v3](https://github.com/aws/aws-sdk-js-v3)).
1. The module overcomes a set of common limitations listed at the bottom of this README.

# Table of Contents

1. [Getting Started](#getting-started)
    1. [Install](#install)
    2. [Code Examples](#code-examples)
        1. [Client initialization](#client-initialization)
        2. [Sync a remote S3 bucket with the local file system](#sync-a-remote-s3-bucket-with-the-local-file-system)
        3. [Sync the local file system with a remote S3 bucket](#sync-the-local-file-system-with-a-remote-s3-bucket)
        4. [Sync two remote S3 buckets](#sync-two-remote-s3-buckets)
        5. [Monitor transfer progress](#monitor-transfer-progress)
        6. [Use AWS SDK command input options](#use-aws-sdk-command-input-options)
        7. [Relocate objects during sync](#relocate-objects-during-sync)
        8. [Filter source files](#filter-source-files)
1. [API Reference](#api-reference)
    - [Class: S3SyncClient](#class-s3-sync-client)
      - [new S3SyncClient(configuration)](#new-s3-sync-client)
      - [client.sync(localDir, bucketUri[, options])](#sync-bucket-with-local)
      - [client.sync(bucketUri, localDir[, options])](#sync-local-with-bucket)
      - [client.sync(sourceBucketUri, targetBucketUri[, options])](#sync-bucket-with-bucket)
1. [Change Log](#change-log)
1. [Comparison with other modules](#comparison-with-other-modules)

## Getting Started

### Install

``npm i s3-sync-client``

### Code Examples

#### Client initialization

``S3SyncClient`` is a wrapper for the AWS SDK ``S3Client`` class.

```javascript
const S3Client = require('@aws-sdk/client-s3');
const S3SyncClient = require('s3-sync-client');

const s3Client = new S3Client({ /* ... */ });
const { sync } = new S3SyncClient({ client: s3Client });
```

#### Sync a remote S3 bucket with the local file system

```javascript
// aws s3 sync /path/to/local/dir s3://mybucket2
await sync('/path/to/local/dir', 's3://mybucket2');
await sync('/path/to/local/dir', 's3://mybucket2', { partSize: 100 * 1024 * 1024 }); // uses multipart uploads for files higher than 100MB

// aws s3 sync /path/to/local/dir s3://mybucket2/zzz --delete
await sync('/path/to/local/dir', 's3://mybucket2/zzz', { del: true });
```

#### Sync the local file system with a remote S3 bucket

```javascript
// aws s3 sync s3://mybucket /path/to/some/local --delete
await sync('s3://mybucket', '/path/to/some/local', { del: true });

// aws s3 sync s3://mybucket2 /path/to/local/dir --dryrun
const syncOps = await sync('s3://mybucket2', '/path/to/local/dir', { dryRun: true });
console.log(syncOps); // log download and delete operations to perform
```

#### Sync two remote S3 buckets

```javascript
// aws s3 sync s3://my-source-bucket s3://my-target-bucket --delete
await sync('s3://my-source-bucket', 's3://my-target-bucket', { del: true });
```

#### Monitor transfer progress

```javascript
const EventEmitter = require('events');
const { TransferMonitor } = require('s3-sync-client');

const monitor = new TransferMonitor();
monitor.on('progress', (progress) => console.log(progress));
setTimeout(() => monitor.abort(), 30000); // optional abort

await sync('s3://mybucket', '/path/to/local/dir', { monitor });

/* output:
...
{
  size: { current: 11925, total: 35688 },
  count: { current: 3974, total: 10000 }
}
...
and aborts unfinished sync after 30s (promise rejected with an AbortError) 
*/

// to pull status info occasionally only, use monitor.getStatus():
const timeout = setInterval(() => console.log(monitor.getStatus()), 2000);
try {
    await sync('s3://mybucket', '/path/to/local/dir', { monitor });
} finally {
    clearInterval(timeout);
}
```

#### Use AWS SDK command input options

```javascript
const mime = require('mime-types');
/*
commandInput properties can either be:
  - fixed values
  - functions, in order to set dynamic values (e.g. using the object key)
*/

// set ACL, fixed value
await sync('s3://mybucket', '/path/to/local/dir', {
    commandInput: {
        ACL: 'aws-exec-read',
    },
});

// set content type, dynamic value (function)
await sync('s3://mybucket1', 's3://mybucket2', {
    commandInput: {
        ContentType: (syncCommandInput) => (
            mime.lookup(syncCommandInput.Key) || 'text/html'
        ),
    },
});
```

#### Relocate objects during sync

```javascript
// sync s3://my-source-bucket/a/b/c.txt to s3://my-target-bucket/zzz/c.txt
await sync('s3://my-source-bucket/a/b/c.txt', 's3://my-target-bucket', {
    relocations: [ // multiple relocations can be applied
        ['a/b', 'zzz'],
    ],
});

// sync s3://mybucket/flowers/red/rose.png to /path/to/local/dir/rose.png
await sync('s3://mybucket/flowers/red/rose.png', '/path/to/local/dir', {
    relocations: [
        ['flowers/red', ''], // folder flowers/red will be flattened during sync
    ],
});
```

Note: relocations are applied after every other options such as filters.

#### Filter source files

```javascript
// aws s3 sync s3://my-source-bucket s3://my-target-bucket --exclude "*" --include "*.txt" --include "flowers/*"
await sync('s3://my-source-bucket', 's3://my-target-bucket', {
    filters: [
        { exclude: () => true }, // exclude everything
        { include: (key) => key.endsWith('.txt') }, // include .txt files
        { include: (key) => key.startsWith('flowers/') }, // also include everything inside the flowers folder
    ],
});
```

Additional code examples are available in the test folder.

## API Reference
<a name="class-s3-sync-client"></a>
### Class: S3SyncClient
<a name="new-s3-sync-client"></a>
#### ``new S3SyncClient(options)``

- `options` *<Object\>*
  - `client` *<S3Client\>* instance of [AWS SDK S3Client](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/index.html).

<a name="sync-bucket-with-local"></a>
#### ``client.sync(localDir, bucketUri[, options])``

- `localDir` *<string\>* Local directory
- `bucketUri` *<string\>* Remote bucket name which may contain a prefix appended with a `/` separator 
- `options` *<Object\>*
  - `commandInput` *<Object\>* Set any of the SDK [*<PutObjectCommandInput\>*](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/interfaces/putobjectcommandinput.html) or [*<CreateMultipartUploadCommandInput\>*](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/interfaces/createmultipartuploadcommandinput.html) options to uploads
  - `del` *<boolean\>* Equivalent to CLI ``--delete`` option
  - `dryRun` *<boolean\>* Equivalent to CLI ``--dryrun`` option
  - `filters` *<Array\>* [Almost](https://github.com/jeanbmar/s3-sync-client/issues/30) equivalent to CLI ``--exclude`` and ``--include`` options. Filters can be specified using plain objects including either an `include` or `exclude` property. The `include` and `exclude` properties are functions that take an object key and return a boolean.
  - `sizeOnly` *<boolean\>* Equivalent to CLI ``--size-only`` option
  - `monitor` *<S3SyncClient.TransferMonitor\>*
    - Attach `progress` event to receive upload progress notifications
    - Emit `abort` event to stop object uploads immediately
  - `maxConcurrentTransfers` *<number\>* Each upload generates a Promise which is resolved when a local object is written to the S3 bucket. This parameter sets the maximum number of upload promises that might be running concurrently.
  - `partSize` *<number\>* Set the part size in **bytes** for multipart uploads. Default to 5 MB.
  - `relocations` *<Array\>* Allows uploading objects to remote folders without mirroring the source directory structure. Each relocation should be specified as an *<Array\>* of `[sourcePrefix, targetPrefix]`.
- Returns: *<Promise\>* Fulfills with an *<Object\>* of sync operations upon success.

Sync a remote S3 bucket with the local file system.  
Similar to AWS CLI ``aws s3 sync localDir bucketUri [options]``.

<a name="sync-local-with-bucket"></a>
#### ``client.sync(bucketUri, localDir[, options])``

- `bucketUri` *<string\>* Remote bucket name which may contain a prefix appended with a ``/`` separator
- `localDir` *<string\>* Local directory
- `options` *<Object\>*
  - `commandInput` *<Object\>* Set any of the SDK [*<GetObjectCommandInput\>*](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/interfaces/getobjectcommandinput.html) options to downloads
  - `del` *<boolean\>* Equivalent to CLI ``--delete`` option
  - `dryRun` *<boolean\>* Equivalent to CLI ``--dryrun`` option
  - `filters` *<Array\>* [Almost](https://github.com/jeanbmar/s3-sync-client/issues/30) equivalent to CLI ``--exclude`` and ``--include`` options. Filters can be specified using plain objects including either an `include` or `exclude` property. The `include` and `exclude` properties are functions that take an object key and return a boolean.
  - `sizeOnly` *<boolean\>* Equivalent to CLI ``--size-only`` option
  - `monitor` *<S3SyncClient.TransferMonitor\>*
    - Attach `progress` event to receive download progress notifications
    - Emit `abort` event to stop object downloads immediately
  - `maxConcurrentTransfers` *<number\>* Each download generates a Promise which is resolved when a remote object is written to the local file system. This parameter sets the maximum number of download promises that might be running concurrently.
  - `relocations` *<Array\>* Allows downloading objects to local directories without mirroring the source folder structure. Each relocation should be specified as an *<Array\>* of `[sourcePrefix, targetPrefix]`.
- Returns: *<Promise\>* Fulfills with an *<Object\>* of sync operations upon success.

Sync the local file system with a remote S3 bucket.  
Similar to AWS CLI ``aws s3 sync bucketUri localDir [options]``.

<a name="sync-bucket-with-bucket"></a>
#### ``client.sync(sourceBucketUri, targetBucketUri[, options])``

- `sourceBucketUri` *<string\>* Remote reference bucket name which may contain a prefix appended with a ``/`` separator
- `targetBucketUri` *<string\>* Remote bucket name to sync which may contain a prefix appended with a ``/`` separator
- `options` *<Object\>*
  - `commandInput` *<Object\>* Set any of the SDK [*<CopyObjectCommandInput\>*](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/interfaces/copyobjectcommandinput.html) options to copy operations
  - `del` *<boolean\>* Equivalent to CLI ``--delete`` option
  - `dryRun` *<boolean\>* Equivalent to CLI ``--dryrun`` option
  - `filters` *<Array\>* [Almost](https://github.com/jeanbmar/s3-sync-client/issues/30) equivalent to CLI ``--exclude`` and ``--include`` options. Filters can be specified using plain objects including either an `include` or `exclude` property. The `include` and `exclude` properties are functions that take an object key and return a boolean.
  - `sizeOnly` *<boolean\>* Equivalent to CLI ``--size-only`` option
  - `monitor` *<S3SyncClient.TransferMonitor\>*
    - Attach `progress` event to receive copy progress notifications
    - Emit `abort` event to stop object copy operations immediately
  - `maxConcurrentTransfers` *<number\>* Each copy generates a Promise which is resolved after the object has been copied. This parameter sets the maximum number of copy promises that might be running concurrently.
  - `relocations` *<Array\>* Allows copying objects to remote folders without mirroring the source folder structure. Each relocation should be specified as an *<Array\>* of `[sourcePrefix, targetPrefix]`.
- Returns: *<Promise\>* Fulfills with an *<Object\>* of sync operations upon success.

Sync two remote S3 buckets.  
Similar to AWS CLI ``aws s3 sync sourceBucketUri targetBucketUri [options]``.

# Change Log

See [CHANGELOG.md](CHANGELOG.md).

# Comparison with other modules

**AWS CLI ``s3 sync`` for Node.js** has been developed to solve the S3 syncing limitations of the existing GitHub repo and NPM modules.

Most of the existing repo and NPM modules suffer one or more of the following limitations:

- requires AWS CLI to be installed
- uses Etag to perform file comparison (Etag should be considered an opaque field and shouldn't be used)
- limits S3 bucket object listing to 1000 objects
- supports syncing bucket with local, but doesn't support syncing local with bucket
- doesn't support multipart uploads
- uses outdated dependencies
- is unmaintained

The following JavaScript modules suffer at least one of the limitations:

- https://github.com/guerrerocarlos/aws-cli-s3-sync
- https://github.com/thousandxyz/s3-lambo
- https://github.com/Quobject/aws-cli-js
- https://github.com/auth0/node-s3-client
- https://github.com/andrewrk/node-s3-client
- https://github.com/hughsk/s3-sync
- https://github.com/issacg/s3sync
