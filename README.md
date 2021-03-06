Audioserve
==========

[![Build Status](https://travis-ci.org/izderadicka/audioserve.svg?branch=master)](https://travis-ci.org/izderadicka/audioserve)

Simple personal server to serve audio files from directories. Intended primarily for audio books, but anything with decent directories structure will do. Focus here is on simplicity and minimalistic design.

Server is in Rust,  default client is in Javascript intended for modern browsers (latest Firefox or Chrome) and is integrated with the server. There is also [Android client](https://github.com/izderadicka/audioserve-android) and API for custom clients.

For some background and video demo check this article [Audioserve Audiobooks Server - Stupidly Simple or Simply Stupid?](http://zderadicka.eu/audioserve-audiobooks-server-stupidly-simple-or-simply-stupid)

Media Library
-------------

Audioserve is intended to serve files from directory in exactly same structure, no audio tags are considered.  So recommended structure is:

    Author Last Name, First Name/Audio Book Name
    Author Last Name, First Name/Series Name/Audio Book Name

Files should be named so they are in right alphabetical order - ideal is:

    001 - First Chapter Name.opus
    002 - Seconf Chapter Name.opus

But this structure is not mandatory -  you will just see whatever directories and files you have, so use anything that will suite you.

In folders you can have additional metadata files - first available image (jpeg or png) is taken as a cover picture and first text file (html, txt, md) is taken as description of the folder.

Search is done for folder names only (not individual files, neither audio metadata tags).

You can have several libraries/ collections - just use several root directories as audioserve start parametes. In client you can switch between collections in the client. Typical usage will be to have separate collections for different languages.

By default symbolic(soft) links are not followed in the collections directory (because if incorretly used it can have quite negative impact on search and browse), but they can be enabled by `--allow-soflinks` program argument.

Security
--------

Audioserve is not writing anything to your media library, so read only access is OK.  The only one file where it needs to write is a file were it keeps its secret key for authentication (by default in `~/.audioserve.secret`, but it can be specified by command line argument). And optionaly it writes files to transcoding cache ([see below](#transcoding-cache)).

Authentication is done by shared secret phrase (supplied to server on command line), which clients must know.
Secret phrase is never sent in plain (it's sent as salted hash). If correct shared secret hash is provided sever generates a token, using its secret key.  Token then can be used in cookie or HTTP Authorization header (Bearer method).
Token validity period is one year by default, but can be set as command line argument, but system generaly expects token validity to be at least 10 days.
As the token can be used to steal the session https is recomended (TLS support is build in).

### TLS/SSL

Audioserve supports TLS/SSL - to enable it you need to provide your private server key as PKCS#12 file (in `--ssl-key` argument). Here is quick tip how to create private key with self-signed certificate:

    openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem \
        -subj "/C=CZ/ST=Prague/L=Prague/O=Ivan/CN=audioserve"
    openssl pkcs12 -inkey key.pem -in certificate.pem -export  -passout pass:mypass -out audioserve.p12
    rm key.pem certificate.pem

You can also run behind reverse proxy like nginx or ha-proxy and perminate SSL there (in that case you can compile audioserve without TLS support see below)

Performance
-----------

Audioserve is inteded to serve personal audio collections of moderate sizes. For sake of simplicity it does not provide any large scale perfomance optimalizations.  It's fine to serve couple of users from collection of couple of thousands audiobooks, if they are reasonably organized. That's it, if you're looking for solution for thousands or millions of users, look elsewere. To compensate for this audioserve is very lightweight and by itself takes minimum of system resources.

Browsing of collections is limited by speed of the file system. As directory listing needs to read audio files metadata (duration and bitrate), folders with too many files (> 200) will be slow. Search is done by walking through collection, it can be slow - especially the first search (subsequent searches are much, much faster, as directory structure from previous search is cached by OS for some time). Recent versions provides possibility for seach cache, to speed up search significantly - [see below](#search-cache).

But true limiting factor is transcoding - as it's quite CPU intensive. Normally you should run only a handful of transcodings in parallel, not much then 2x - 4x more then there is the number of cores in the machine. For certain usage scenarios enabling of [transcoding cache](#transcoding-cache) can help a bit.

### Search Cache

For fast searches enable search cache with `--search-cache`, it will load directory structure of collections into memory, so searches will be blazingly fast (for price of more occupied memory). Search cache monitors directories and update itself upon changes (make take a while). Also after start of audioserve it takes some time before cache is filled (especially when large collections are used), so search might not work initially.

### Transcoding Cache

Optionally you can enable transcoding cache (by compiling audioserve with transcoding-cache feature). Contribution of this cache to overall performance depends very much on usage scenarios.  If there is only one user, which basically listens to audiobooks in linear order (chapter after chapter, not jumping back and forth), benefit will be minimal. If there are more users, listening to same audiobook (with same transcoding levels) and/or jumping often back and forth between chapters, then benefit of this cache can be significat. You should test to see the difference (when transcoding cache is compiled in it can be still disabled by `--t-cache-disable` option).

Transcoding
-----------

Audioserve offers possibility to transcode audio files to opus format (opus codec, ogg or webm container) to save bandwidth and volume of transfered data. For transcoding to work `ffmpeg` program must be installed and available on system's PATH.
Transconding is provided in three variants and client can choose between then (using query parameter trans with value l,m or h):

* low - (default 32 kbps opus with 12kHz cutoff)
* medium - (default 48 kbps opus with 12kHz cutoff)
* high - (default 64 kbps opus with 20kHz cutoff)

As already noted audioserve is intended primarily for audiobooks and believe me opus codec is excellent there even in low bitrates. However if you want to change parameters of these three trancodings you can easily do so by providing yaml confing file to argument `--transcoding-config`. Here is sample file:

```yaml
low:
  opus-in-ogg:
    bitrate: 16
    compression_level: 3
    cutoff: WideBand
medium:
  opus-in-ogg:
    bitrate: 24
    compression_level: 6
    cutoff: SuperWideBand
high:
  opus-in-ogg:
    bitrate: 32
    compression_level: 9
    cutoff: SuperWideBand
```

In each key first you have specification of codec-container combination, currently it supports `opus-in-ogg`, `opus-in-webm`, `mp3`, `aac-in-adts` (but other containers or codecs can relatively easily added, provided they are supported by ffmpeg and container creation does not require seekable output - like MP4 container).

I have good experinces with `opus-in-ogg`, which is also default. `opus-in-webm` works well in browsers (and is supported  in browsers MSE API), but as it does not contain audio duration after trascoding, audio cannot be sought during playback in Android client, which is significant drawback. `mp3` is classical MPEG-2 Audio Layer III audio stream. It has three parametes. `aac-in-adts` AAC encoded audio in ADTS stream, similarly as `opus-in-webm` it has problems with seeking in Android client.

For opus transcodings there are 3 other parameters, where `bitrate` is desired bitrate in kbps, `compression_level` is determining audio quality and speed of transcoding with values 1-10 ( 1 - worst quality, but fastest, 10 - best quality, but slowest ) and `cutoff` is determining audio freq. bandwith (NarrowBand => 4kHz, MediumBand => 6kHz, WideBand => 8kHz, SuperWideBand => 12kHz, FullBand => 20kHz).

For mp3 transcoding there are also 3 parameters: `bitrate` (in kbps), `compression_level` with values 0-9 (0 - best quality, slowest, 9-worst quality, fastest; so meaning is inverse then for opus codec) and `abr` (optional), which can be `true` or `false` (ABR = average bit rate - enables ABR, which is similar to VBR, so it can improve quality on same bitrate, however can cause problems, when seeking in audion stream).

`aac-in-adts` has one mandatory parameter `bitrate` (in kbps) and two optional parameters `sr` - which is sample rate of transcoded stream (8kHz, 12kHz, 16kHz, 24kHz, 32kHz, 48kHz, unlimited) and `ltp` (Long Term Prediction), which is `true` or `false` and can improve audio quality, especially for lower bitrates, but for significant performance costs ( abou 10x slower).

Overall `opus-in-ogg` provides best results from both quality and  functionality perspective, so I'd highly recommend to stick to it, unless you have some problem with it.

You can overide one two or all three defaults, depending on what sections you have in this config file.

Command line
------------

Check with `audioserve -h`. Only two required arguments are shared secrect and root of media library (as noted above you can have severals libraries).
`audioserve`  is server executable and it also needs web client files , which are `index.html` and `bundle.js`, which are defaultly in `./client/dist`, but their location can by specified by argument `-c`.

Android client
--------------

Android client code is [available on github](https://github.com/izderadicka/audioserve-android)
Client is in early beta stage (I'm using it now to listen to my audiobooks).

API
---

audioserve server provides very simple API (see [api.md](./docs/api.md) for documentation), so it's easy to write your own clients.

Installation (Linux)
------------

Install required dependencies:

    # Ubuntu - for other distros look for equivalent packages
    sudo apt-get install -y  openssl libssl-dev libtag1-dev libtagc0-dev ffmpeg

Clone repo with:

    git clone https://github.com/izderadicka/audioserve

To install locally you need [Rust](https://www.rust-lang.org/en-US/install.html) and [NodeJS](https://nodejs.org/en/download/package-manager/) installed - compile with `cargo build --release` (Rust code have system dependencies to openssl and taglib) and build client in its directory:

    npm install
    npm run build

But easiest way how to test audioserve is to run it as docker container with provided `Dockerfile`, just run:

    docker build --tag audioserve .
    docker run -d --name audioserve -p 3000:3000 -v /path/to/your/audiobooks:/audiobooks  audioserve  

Then open <https://localhost:3000> and accept insecure connection, shared secret to enter in client is mypass

The following environment variables can be used to customise how audioserve runs:

    DIRS - Space separated list of audio folders (defaults to: /audiobooks)
    SECRET - The shared secret key (defaults to: mypass)
    SSLKEY - Path to the SSL key (defaults to: ./ssl/audioserve.p12)
    SSLPASS - Password to the SSL key (defaults to: mypass)
    OTHER_ARGS - Any of the other audioserve command line options, such as --search-cache, space separated

Note: If you do not wish to use the SSL certificate, blank values for SSLKEY and SSLPASS will suffice.

A more detailed example:

    docker run -d --name audioserve -p 3000:3000 \
        -v /path/to/your/audiobooks1:/collection1 \
        -v /path/to/your/audiobooks2:/collection2 \
        -e DIRS=/collection1 /collection2 \
        -e SECRET=mypassword \
        -e SSLKEY= \
        -e SSLPASS= \
        -e OTHER_ARGS=--search-cache \
        audioserve

In the above example, we are adding two different collections of audiobooks (collection1 and collection2).
Both are made available to the container via -v option and passed to audioserve in the -e DIRS ENV variable.
We set the shared secret via -e SECRET and specify the use of a search cache via -e OTHER_ARGS.
In this case, we are not using SSL and this is determined via the blank values in -e SSLKEY and -e SSLPASS.

Other platforms - theoretically audioserve can work on Windows and MacOS (probably with few changes),
but I never tried to build it there. Any help in this area is welcomed.

### Compiling without default features or with non-default features

TLS support, symbolic links and search cache are default features, but you can compile without them - just add `--no-default-features` option to cargo. And then evetually choose only features you need. To add non-default features (like transcoding-cache) compile with this option `--features transcoding-cache` in `cargo build` command. Or you can use option `--all-features` to enable all available features.

License
-------

[MIT](https://opensource.org/licenses/MIT)
