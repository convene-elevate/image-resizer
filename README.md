# Image-Resizer

This is a modified version of the official npm package `image-resizer` found here: https://www.npmjs.com/package/image-resizer
This documentation has been modified to denote the main changes and only include relevant information. 

Specifically, this version has been modified so that image modifers are set via URL query string parameters, rather than being set in the URL path.

## Overview

`image-resizer` uses [sharp](https://github.com/lovell/sharp) under the hood to modify and optimise your images.

There is also a plugin architecture that allows you to add your own image sources. Out of the box it supports: S3, Facebook, Twitter, Youtube, Vimeo (and local file system in development mode).

When a new image size is requested of `image-resizer` via the CDN, it will pull down the original image from the cloud. It will then resize according to the requested dimensions, optimize according to file type and optionally filter the image. All responses are crafted with custom responses to maximise the facility of the CDN.

## Dependencies

`image-resizer` only requires a working node/npm environment and `libvips`. 

## Environment Variables

Configuration of `image-resizer` is done via environment variables.

The minimum required variables are:

    AWS_ACCESS_KEY_ID
    AWS_SECRET_ACCESS_KEY
    AWS_REGION
    S3_BUCKET
    NODE_ENV

If you choose to change your default source to be something other than `S3` then the `NODE_ENV` variable is the only required one (and whatever you need for your default source).

For convenience in local and non-Heroku deployments the variables can be loaded from a `.env` file. Sensible local defaults are included in `src/config/environment_vars.js`.

The available variables are as follows:

```javascript
  NODE_ENV: 'development',
  PORT: 3001,
  DEFAULT_SOURCE: 's3',
  EXCLUDE_SOURCES: null, // add comma delimited list

  // Restrict to named modifiers strings only
  NAMED_MODIFIERS_ONLY: false,

  // AWS keys
  AWS_ACCESS_KEY_ID: null,
  AWS_SECRET_ACCESS_KEY: null,
  AWS_REGION: null,
  S3_BUCKET: null,

  // Resize options
  RESIZE_PROCESS_ORIGINAL: true,
  AUTO_ORIENT: true,
  REMOVE_METADATA: true,

  // Protect original files by specifying a max image width or height - limits
  // max height/width in parameters
  MAX_IMAGE_DIMENSION: null,

  // Color used when padding an image with the 'pad' crop modifier.
  IMAGE_PADDING_COLOR: 'white',

  // Optimization options
  IMAGE_QUALITY: 80,
  IMAGE_PROGRESSIVE: true,

  // Cache expiries
  IMAGE_EXPIRY: 60 * 60 * 24 * 90,
  IMAGE_EXPIRY_SHORT: 60 * 60 * 24 * 2,
  JSON_EXPIRY: 60 * 60 * 24 * 30,

  // Logging options
  LOG_PREFIX: 'resizer',
  QUEUE_LOG: true,

  // Response settings
  CACHE_DEV_REQUESTS: false,

  // Twitter settings
  TWITTER_CONSUMER_KEY: null,
  TWITTER_CONSUMER_SECRET: null,
  TWITTER_ACCESS_TOKEN: null,
  TWITTER_ACCESS_TOKEN_SECRET: null,

  // Where are the local files kept?
  LOCAL_FILE_PATH: process.cwd(),

  // Display an image if a 404 request is encountered from a source
  IMAGE_404: null

  // Whitelist arbitrary HTTP source prefixes using EXTERNAL_SOURCE_*
  EXTERNAL_SOURCE_WIKIPEDIA: 'https://upload.wikimedia.org/wikipedia/'
```

## Optimization

Optimization of images is done via [sharp](https://github.com/lovell/sharp#qualityquality). The environment variables to set are:

* `IMAGE_QUALITY`:  1 - 100
* `IMAGE_PROGRESSIVE`:  true | false

You may also adjust the image quality setting per request with the `q` quality modifier described below.

## Usage

A couple of routes are included with the default app, but the most important is the image generation one, which is as follows:

`http://my.cdn.com/path/to/image.png[:format][:metadata]?{modifiers}`

Modifiers are query string parameters of the requested modifications to be made, they include:

*Supported modifiers are:*
* height:       eg. h=500
* width:        eg. w=200
* square:       eg. s=50
* crop:         eg. c=fill
* top:          eg. y=12
* left:         eg. x=200
* gravity:      eg. g=s, g=ne
* filter:       eg. f=sepia
* external:     eg. e=facebook
* quality:      eg. q=90

*Crop modifiers:*
* fit
    * maintain original proportions
    * resize so image fits wholly into new dimensions
        * eg: h400-w500 - 400x600 -> 333x500
    * default option
* fill
    * maintain original proportions
    * resize via smallest dimension, crop the largest
    * crop image all dimensions that dont fit
        * eg: h400-w500 - 400x600 -> 400x500
* cut
    * maintain original proportions
    * no resize, crop to gravity or x/y
* scale
    * do not maintain original proportions
    * force image to be new dimensions (squishing the image)
* pad
    * maintain original proportions
    * resize so image fits wholly into new dimensions
    * padding added on top/bottom or left/right as needed (color is configurable)


*Examples:*
* `http://my.cdn.com/path/to/image.png?s=50`
* `http://my.cdn.com/path/to/image.png?h=50`
* `http://my.cdn.com/path/to/image.png?h=50&w=100`
* `http://my.cdn.com/path/to/image.png?s=50&g=ne`
* `http://my.cdn.com/path/to/image.png` - original image request, will be optimized but not resized


## Resizing Logic

It is worthy of note that this application will not scale images up, we are all about keeping images looking good. So a request for `?h=400` on an image of only 200px in height will not scale it up.


## S3 source

By default `image-resizer` will use s3 as the image source. To access an s3 object the full path of the image within the bucket is used, minus the bucket name eg:

    https://s3.amazonaws.com/sample.bucket/test/image.png

translates to:

    http://my.cdn.com/test/image.png


## External Sources

It is possible to bring images in from external sources and store them behind your own CDN. This is very useful when it comes to things like Facebook or Vimeo which have very inconsistent load times. Each external source can still enable any of the modification parameters list above.

In addition to the provided external sources, you can easily add your own basic external sources using `EXTERNAL_SOURCE_*` environment variables. For example, to add Wikipedia as an external source, set the following environment variable:

```
EXTERNAL_SOURCE_WIKIPEDIA: 'https://upload.wikimedia.org/wikipedia/'
```

Then you can request images beginning with the provided path using the `ewikipedia` modifier, eg:

    http://my.cdn.com/ewikipedia/en/7/70/Example.png
    
translates to:

    https://upload.wikimedia.org/wikipedia/en/7/70/Example.png

It is worth noting that Twitter requires a full set of credentials as you need to poll their API in order to return profile pics.

A shorter expiry on images from social sources can also be set via `IMAGE_EXPIRY_SHORT` env var so they expiry at a faster rate than other images.

It is also trivial to write new source streams via the plugins directory. Examples are in `src/streams/sources/`.

## Output format

You can convert images to another image format by appending an extra extension to the image path:

* `http://my.cdn.com/path/to/image.png.webp`

JPEG (`.jpg`/`.jpeg`), PNG (`.png`), and WEBP (`.webp`) output formats are supported.

## Metadata requests

`image-resizer` can return the image metadata as a json endpoint:

* `http://my.cdn.com/path/to/image.png.json`

Metadata is removed in all other image requests by default, unless the env var `REMOVE_METADATA` is set to `false`.
