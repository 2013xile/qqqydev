---
title: Packaging and downloading files through streaming in NodeJS
date: "2023-09-21"
categories:
  - "Backend"
  - "NodeJS"
---

# Background

Last week, I developed a feature that can package the log files produced by the application into a `tar.gz` file and enable online downloading. While this is may seem like a straight forward feature, there are still some intriguing aspects worth documenting.

# Generate temporary file, delete later

The simplest method to implement this feature is to generate a tarball on the server's disk and delete it after it's been downloaded. The code is roughly like this:

```JavaScript
import fs from 'fs';
import tar from 'tar';
import { unlink } from 'fs/promises';

async function dowload(ctx, next) {
  const { files } = ctx.request.body;
  const file = 'logs.tar.gz';
  try {
    ctx.attachment(file);
    await tar.c({
      gzip: true,
      file,
    }, files);
    ctx.body = fs.createReadStream(__dirname + `/${file}`);
    ctx.body.on('end', async () => await unlink(file));
  } catch (err) {
    console.log(err);
  }
  await next();
}
```

> **Note**  
> Since we use [Koa](https://koajs.com/), I can make use of `ctx.attachment`. It is equivalent to:

```JavaScript
this.set('Content-disposition', 'attachment; filename=' + filename);
this.set('Content-type', 'application/gzip');
```

# Stream-based packaging and download

Creating a temporary file and deleting it after download seems straightforward, but it can be considered cumbersome or less elegant since it still occupies disk space temporarily and may lead to issues if errors occur during this process and potential risk of leaving the file behind if deletion fails.

A common alternative is packaging files via streaming, demonstrated by the following code:

```TypeScript
const tarFile = (files: string[]) {
  return new Promise((resolve, reject) => {
    const passthrough = new stream.PassThrougth();
    tar.c({
      gzip: true,
    }, files)
    .pipe(passthrough)
    .on('end', () => {
      passthrough.end();
      resolve(passthrough);
    })
    .on('error', (err) => reject(err));
  });
}

async function dowload(ctx, next) {
  const { files } = ctx.request.body;
  try {
    ctx.attachment('logs.tar.gz');
    ctx.body = await tarFile(files);
  } catch (err) {
    console.log(err);
  }
  await next();
}
```

Here we use `stream.PassThrough` to establish a stream as an intemediary for passing the input bytes across to the output.

# Break data into chunks

Well, the code above seems to work fine initially. However, in production, it hangs when attempting to
package and download large log files. I guess I should chunk the data during packaging due to memory limitations.

```TypeScript
const tarFile = (files: string[]) {
  return new Promise((resolve, reject) => {
    const passthrough = new stream.PassThrougth();
    tar.c({
      gzip: true,
    }, files)
    .on('data', (chunk) => {
      passthrough.write(chunk);
    })
    .on('end', () => {
      passthrough.end();
      resolve(passthrough);
    })
    .on('error', (err) => reject(err));
  });
}
```

# `node-tar` vs `tar-fs`

Another issue arises while tring to download a changing online log file, resulting the "Error: did not encounter expected EOF" error throw by the package `node-tar`. The only reference I have is this issue: [https://github.com/isaacs/node-tar/issues/238](https://github.com/isaacs/node-tar/issues/238). I realized this is a feature and upon inspecting the source code, I couldn't find a method to change this behavior without modifying the source code.

I had to switch to another package, `tar-fs`, for packaging, and it work seamlessly. While it does not offer a built-in gunzip feature, we have a native Node package called `zlib` that can be utilized. The final code:

```TypeScript
const tarFiles = (files: string[]): Promise<any> => {
  return new Promise((resolve, reject) => {
    const passthrough = new stream.PassThrough();
    const gz = zlib.createGzip();
    pack(getLoggerFilePath(), {
      entries: files,
    })
      .on('data', (chunk) => {
        passthrough.write(chunk);
      })
      .on('end', () => {
        passthrough.end();
      })
      .on('error', (err) => reject(err));
    passthrough
      .on('data', (chunk) => {
        gz.write(chunk);
      })
      .on('end', () => {
        gz.end();
        resolve(gz);
      })
      .on('error', (err) => reject(err));
    gz.on('error', (err) => reject(err));
  });
};
```
