# [Shrine]

Shrine is a toolkit for file attachments in Ruby applications. Some highlights:

* **Modular design** – the [plugin system] allows you to load only the functionality you need
* **Memory friendly** – streaming uploads and [downloads][Retrieving Uploads] make it work great with large files
* **Cloud storage** – store files on [disk][FileSystem], [AWS S3][S3], [Google Cloud][GCS], [Cloudinary] and [others][external]
* **ORM integrations** – works with [Sequel][sequel plugin], [ActiveRecord][activerecord plugin], [Hanami::Model][hanami plugin] and [Mongoid][mongoid plugin]
* **Flexible processing** – generate thumbnails [on upload] or [on-the-fly] using [ImageMagick][ImageProcessing::MiniMagick] or [libvips][ImageProcessing::Vips]
* **Metadata validation** – [validate files][validation] based on [extracted metadata][metadata]
* **Direct uploads** – upload asynchronously [to your app][simple upload] or [to the cloud][presigned upload] using [Uppy]
* **Resumable uploads** – make large file uploads [resumable][resumable upload] on [S3][uppy-s3_multipart] or [tus][tus-ruby-server]
* **Background jobs** – built-in support for [background processing][backgrounding] that supports [any backgrounding library][backgrounding libraries]

If you're curious how it compares to other file attachment libraries, see the [Advantages of Shrine].

## Resources

| Resource          | URL                                                                                        |
| :---------------- | :----------------------------------------------------------------------------------------- |
| Website           | [shrinerb.com](https://shrinerb.com)                                                       |
| Demo code         | [Roda][roda demo] / [Rails][rails demo]                                                    |
| Source            | [github.com/shrinerb/shrine](https://github.com/shrinerb/shrine)                           |
| Wiki              | [github.com/shrinerb/shrine/wiki](https://github.com/shrinerb/shrine/wiki)                 |
| Bugs              | [github.com/shrinerb/shrine/issues](https://github.com/shrinerb/shrine/issues)             |
| Help & Discussion | [groups.google.com/group/ruby-shrine](https://groups.google.com/forum/#!forum/ruby-shrine) |

## Contents

* [Quick start](#quick-start)
* [Storage](#storage)
* [Uploader](#uploader)
  - [Uploading](#uploading)
  - [IO abstraction](#io-abstraction)
* [Uploaded file](#uploaded-file)
* [Attachment](#attachment)
* [Attacher](#attacher)
* [Plugin system](#plugin-system)
* [Metadata](#metadata)
  * [MIME type](#mime-type)
  * [Other metadata](#other-metadata)
* [Processing](#processing)
  * [Processing on upload](#processing-on-upload)
  * [Processing on-the-fly](#processing-on-the-fly)
* [Validation](#validation)
* [Context](#context)
* [Location](#location)
* [Direct uploads](#direct-uploads)
  - [Simple direct upload](#simple-direct-upload)
  - [Presigned direct upload](#presigned-direct-upload)
  - [Resumable direct upload](#resumable-direct-upload)
* [Backgrounding](#backgrounding)
* [Clearing cache](#clearing-cache)
* [Logging](#logging)
* [Settings](#settings)

## Quick start

Add Shrine to the Gemfile and write an initializer which sets up the storage and
loads the ORM plugin:

```rb
# Gemfile
gem "shrine", "~> 2.0"
```

```rb
require "shrine"
require "shrine/storage/file_system"

Shrine.storages = {
  cache: Shrine::Storage::FileSystem.new("public", prefix: "uploads/cache"), # temporary
  store: Shrine::Storage::FileSystem.new("public", prefix: "uploads"),       # permanent
}

Shrine.plugin :sequel # or :activerecord
Shrine.plugin :cached_attachment_data # for retaining the cached file across form redisplays
Shrine.plugin :restore_cached_data # re-extract metadata when attaching a cached file
Shrine.plugin :rack_file # for non-Rails apps
```

Next decide how you will name the attachment attribute on your model, and run a
migration that adds an `<attachment>_data` text or JSON column, which Shrine
will use to store all information about the attachment:

```rb
Sequel.migration do
  change do
    add_column :photos, :image_data, :text
  end
end
```

In Rails with ActiveRecord the migration would look similar:

```sh
$ rails generate migration add_image_data_to_photos image_data:text
```
```rb
class AddImageDataToPhotos < ActiveRecord::Migration
  def change
    add_column :photos, :image_data, :text
  end
end
```

Now you can create an uploader class for the type of files you want to upload,
and add a virtual attribute for handling attachments using this uploader to
your model. If you do not care about adding plugins or additional processing,
you can use `Shrine::Attachment`.

```rb
class ImageUploader < Shrine
  # plugins and uploading logic
end
```

```rb
class Photo < Sequel::Model # ActiveRecord::Base
  include ImageUploader::Attachment.new(:image) # adds an `image` virtual attribute
end
```

Let's now add the form fields which will use this virtual attribute. We need
(1) a file field for choosing files, and (2) a hidden field for retaining the
uploaded file in case of validation errors and for potential [direct
uploads].

```rb
# with Rails form builder:
form_for @photo do |f|
  f.hidden_field :image, value: @photo.cached_image_data
  f.file_field :image
  f.submit
end
```
```rb
# with Simple Form:
simple_form_for @photo do |f|
  f.input :image, as: :hidden, input_html: { value: @photo.cached_image_data }
  f.input :image, as: :file
  f.button :submit
end
```
```rb
# with Forme:
form @photo, action: "/photos", enctype: "multipart/form-data" do |f|
  f.input :image, type: :hidden, value: @photo.cached_image_data
  f.input :image, type: :file
  f.button "Create"
end
```

Note that the file field needs to go *after* the hidden field, so that
selecting a new file can always override the cached file in the hidden field.
Also notice the `enctype="multipart/form-data"` HTML attribute, which is
required for submitting files through the form (the Rails form builder
will automatically generate this for you).

Now in your router/controller the attachment request parameter can be assigned
to the model like any other attribute:

```rb
# In Rails:
class PhotosController < ApplicationController
  def create
    Photo.create(params[:photo])
    # ...
  end
end
```
```rb
# In Sinatra:
post "/photos" do
  Photo.create(params[:photo])
  # ...
end
```

Once a file is uploaded and attached to the record, you can retrieve a URL to
the uploaded file with `#<attachment>_url` and display it on the page:

```erb
<!-- In Rails: -->
<%= image_tag @photo.image_url %>
```
```erb
<!-- In HTML: -->
<img src="<%= @photo.image_url %>" />
```

## Storage

A "storage" in Shrine is an object that encapsulates communication with a
specific storage service, by implementing a common public interface. Storage
instances are registered under an identifier in `Shrine.storages`, so that they
can later by used by [uploaders][uploader].

Previously we've shown the [FileSystem] storage which saves files to disk, but
Shrine also ships with [S3] storage which stores files on [AWS S3] (or any
S3-compatible service such as [DigitalOcean Spaces] or [MinIO]).

```rb
# Gemfile
gem "aws-sdk-s3", "~> 1.14" # for AWS S3 storage
```
```rb
require "shrine/storage/s3"

s3_options = {
  bucket:            "<YOUR BUCKET>", # required
  access_key_id:     "<YOUR ACCESS KEY ID>",
  secret_access_key: "<YOUR SECRET ACCESS KEY>",
  region:            "<YOUR REGION>",
}

Shrine.storages = {
  cache: Shrine::Storage::S3.new(prefix: "cache", **s3_options),
  store: Shrine::Storage::S3.new(**s3_options),
}
```

The above example sets up S3 for both temporary and permanent storage, which is
suitable for [direct uploads][direct S3 uploads guide]. The `:cache` and
`:store` names are special only in terms that the [attacher] will automatically
pick them up, you can also register more storage objects under different names.

See the [FileSystem] and [S3] storage docs for more details. There are [many
more Shrine storages][external] provided by external gems, and you can also
[create your own storage][Creating Storages].

## Uploader

Uploaders are subclasses of `Shrine`, and they wrap the actual upload to the
storage. They perform common tasks around upload that aren't related to a
particular storage.

```rb
class MyUploader < Shrine
  # image attachent logic
end
```

It's common to create an uploader for each type of file that you want to handle
(`ImageUploader`, `VideoUploader`, `AudioUploader` etc), but really you can
organize them in any way you like.

### Uploading

The main method of the uploader is `#upload`, which takes an [IO-like
object][io abstraction] and a storage identifier on the input, and returns a
representation of the [uploaded file] on the output.

```rb
MyUploader.upload(file, :store) #=> #<Shrine::UploadedFile>
```

Internally this instantiates the uploader with the storage and calls `#upload`
on it:

```rb
uploader = MyUploader.new(:store)
uploader.upload(file) #=> #<Shrine::UploadedFile>
```

Some of the tasks performed by `#upload` include:

* any defined [file processing][on upload]
* extracting [metadata]
* generating [location]
* uploading (this is where the [storage] is called)
* closing the uploaded file

Additional upload options can be passed via `:upload_options`, and they will be
forwarded directly to `Storage#upload` (see the documentation of your storage
for the list of available options):

```rb
uploader.upload(file, upload_options: { acl: "public-read" })
```

### IO abstraction

Shrine is able to upload any IO-like object that responds to `#read`,
`#rewind`, `#eof?` and `#close`. This includes built-in IO and IO-like objects
like File, Tempfile and StringIO.

When a file is uploaded to a Rails app, it will be represented by an
`ActionDispatch::Http::UploadedFile` object in the params. This is also an
IO-like object accepted by Shrine. In other Rack applications the uploaded file
will be represented as a Hash, but it can be converted into an IO-like object
with the [`rack_file`][rack_file plugin] plugin.

Here are some examples of various IO objects that can be uploaded:

```rb
uploader.upload File.open("/path/to/file", binmode: true)   # upload from disk
uploader.upload StringIO.new("file content")                # upload from memory
uploader.upload ActionDispatch::Http::UploadedFile.new(...) # upload from Rails controller
uploader.upload Shrine.rack_file({ tempfile: tempfile })    # upload from Rack controller
uploader.upload Rack::Test::UploadedFile.new(...)           # upload from rack-test
uploader.upload Down.open("https://example.org/file")       # upload from internet
```

`Shrine::UploadedFile`, the object returned after upload, is itself an IO-like
object as well. This makes it trivial to re-upload a file from one storage to
another, and this is used by the [attacher] to copy files from temporary to
permanent storage.

## Uploaded file

The `Shrine::UploadedFile` object represents the file that was uploaded to the
storage, and it's what's returned from `Shrine#upload` or when retrieving a
record attachment. It contains the following information:

| Key        | Description                                        |
| :-------   | :----------                                        |
| `id`       | location of the file on the storage                |
| `storage`  | identifier of the storage the file was uploaded to |
| `metadata` | file [metadata] that was extracted before upload   |

```rb
uploaded_file = uploader.upload(file)

uploaded_file.id       #=> "949sdjg834.jpg"
uploaded_file.storage  #=> #<Shrine::Storage::FileSystem>
uploaded_file.metadata #=> {...}

# It can be serialized into JSON and saved to a database column
uploaded_file.to_json  #=> '{"id":"949sdjg834.jpg","storage":"store","metadata":{...}}'
```

It comes with many convenient methods that delegate to the storage:

```rb
uploaded_file.url                 #=> "https://my-bucket.s3.amazonaws.com/949sdjg834.jpg"
uploaded_file.open                # opens the uploaded file
uploaded_file.download            #=> #<File:/var/folders/.../20180302-33119-1h1vjbq.jpg>
uploaded_file.stream(destination) # streams uploaded content into a writable destination
uploaded_file.exists?             #=> true
uploaded_file.delete              # deletes the file from the storage

# open/download the uploaded file for the duration of the block
uploaded_file.open     { |io| io.read }
uploaded_file.download { |tempfile| tempfile.read }
```

It also implements the IO-like interface that conforms to Shrine's [IO
abstraction][io abstraction], which allows it to be uploaded again to other
storages.

```rb
uploaded_file.read   # returns content of the uploaded file
uploaded_file.eof?   # returns true if the whole IO was read
uploaded_file.rewind # rewinds the IO
uploaded_file.close  # closes the IO
```

For more details, see the [Retrieving Uploads] guide and
[`Shrine::UploadedFile`] API docs.

## Attachment

Storages, uploaders, and uploaded file objects are Shrine's foundational
components. To help you actually attach uploaded files to database records in
your application, Shrine comes with a high-level attachment interface, which
is built on top of these components.

Shrine has plugins for hooking into most database libraries, and in case of
ActiveRecord and Sequel the plugin will automatically tie the attached files to
records' lifecycles. You can also use Shrine just with plain old Ruby objects.

```rb
Shrine.plugin :sequel # :activerecord
```

```rb
class Photo < Sequel::Model # ActiveRecord::Base
  include ImageUploader::Attachment.new(:image) #
  include ImageUploader::Attachment(:image)     # these are all equivalent
  include ImageUploader[:image]                 #
end
```

You can choose whichever of these syntaxes you prefer. Either of these
will create a `Shrine::Attachment` module with attachment methods for the
specified attribute, which then get added to your model when you include it:

| Method            | Description                                                                       |
| :-----            | :----------                                                                       |
| `#image=`         | uploads the file to temporary storage and serializes the result into `image_data` |
| `#image`          | returns [`Shrine::UploadedFile`][uploaded file] instantiated from `image_data`    |
| `#image_url`      | calls `url` on the attachment if it's present, otherwise returns nil              |
| `#image_attacher` | returns instance of [`Shrine::Attacher`][attacher] which handles the attaching    |

The ORM plugin that we loaded adds appropriate callbacks. For example, saving
the record uploads the attachment to permanent storage, while deleting the
record deletes the attachment.

```rb
# no file is attached
photo.image #=> nil

# the assigned file is cached to temporary storage and written to `image_data` column
photo.image = File.open("waterfall.jpg")
photo.image      #=> #<Shrine::UploadedFile @data={...}>
photo.image_url  #=> "/uploads/cache/0sdfllasfi842.jpg"
photo.image_data #=> '{"id":"0sdfllasfi842.jpg","storage":"cache","metadata":{...}}'

# the cached file is promoted to permanent storage and saved to `image_data` column
photo.save
photo.image      #=> #<Shrine::UploadedFile @data={...}>
photo.image_url  #=> "/uploads/store/l02kladf8jlda.jpg"
photo.image_data #=> '{"id":"l02kladf8jlda.jpg","storage":"store","metadata":{...}}'

# the attached file is deleted with the record
photo.destroy
photo.image.exists? #=> false
```

If there is already a file attached and a new file is attached, the previous
attachment will get deleted when the record gets saved.

```rb
photo.update(image: new_file) # changes the attachment and deletes previous
# or
photo.update(image: nil)      # removes the attachment and deletes previous
```

In addition to assigning raw files, you can also assign a JSON representation
of files that are already uploaded to the temporary storage. This allows Shrine
to retain cached files in case of validation errors and handle [direct uploads]
via the hidden form field.

```rb
photo.image = '{"id":"9260ea09d8effd.jpg","storage":"cache","metadata":{...}}'
```

## Attacher

The model attachment attributes and callbacks just delegate the behaviour
to their underlying `Shrine::Attacher` object.

```rb
photo.image_attacher #=> #<Shrine::Attacher>
```

The `Shrine::Attacher` object can be instantiated and used directly:

```rb
attacher = ImageUploader::Attacher.new(photo, :image)

attacher.assign(file) # equivalent to `photo.image = file`
attacher.get          # equivalent to `photo.image`
attacher.url          # equivalent to `photo.image_url`
```

The attacher is what drives attaching files to model instances, and it functions
independently from models' attachment interface. This means that you can use it
as an alternative, in case you prefer not to add additional attributes to the
model, or prefer explicitness over callbacks. It's also useful when you need
something more advanced which isn't available through the attachment
methods.

The `Shrine::Attacher` by default uses `:cache` for temporary and `:store` for
permanent storage, but you can specify a different storage:

```rb
ImageUploader::Attacher.new(photo, :image, cache: :other_cache, store: :other_store)

# OR

photo.image_attacher(cache: :other_cache, store: :other_store)
photo.image = file # uploads to :other_cache storage
photo.save         # promotes to :other_store storage
```

You can also skip the temporary storage altogether and upload files directly to
the primary storage:

```rb
uploaded_file = attacher.store!(file) # upload file directly to permanent storage
attacher.set(uploaded_file)           # attach the uploaded file
```

Whenever the attacher uploads or deletes files, it sends a [context
hash][context] which includes `:record`, `:name`, and `:action` keys, so that
you can perform processing or generate location differently depending on this
information.

For more information about `Shrine::Attacher`, see the [Using Attacher] guide.

## Plugin system

By default Shrine comes with a small core which provides only the essential
functionality. All additional features are available via [plugins], which also
ship with Shrine. This way you can choose exactly what and how much Shrine does
for you, and you load the code only for features that you use.

```rb
Shrine.plugin :logging # adds logging
```

Plugins add behaviour by extending Shrine core classes via module inclusion, and
many of them also accept configuration options. The plugin system respects
inheritance, so you can choose to load a plugin globally or per uploader.

```rb
class ImageUploader < Shrine
  plugin :store_dimensions # extract image dimensions only for this uploader and its descendants
end
```

If you want to extend Shrine functionality with custom behaviour, you can also
[create your own plugin][Creating Plugins].

## Metadata

Shrine automatically extracts some basic file metadata and saves them to the
`Shrine::UploadedFile`. You can access them through the `#metadata` hash or via
metadata methods:

```rb
uploaded_file.metadata #=>
# {
#   "filename" => "matrix.mp4",
#   "mime_type" => "video/mp4",
#   "size" => 345993,
# }

uploaded_file.original_filename #=> "matrix.mp4"
uploaded_file.extension         #=> "mp4"
uploaded_file.mime_type         #=> "video/mp4"
uploaded_file.size              #=> 345993
```

By default these values are determined from the following attributes on the IO
object:

| Key         | Default source                                     |
| :-----      | :------                                            |
| `filename`  | extracted from `io.original_filename` or `io.path` |
| `mime_type` | extracted from `io.content_type`                   |
| `size`      | extracted from `io.size`                           |

### MIME type

By default, `mime_type` will be inherited from `#content_type` attribute of the
uploaded file, which is set from the `Content-Type` request header. However,
this header is determined by the browser solely based on the file extension.
This means that by default Shrine's `mime_type` metadata is *not guaranteed* to
hold the actual MIME type of the file.

To remedy that, you can load the [`determine_mime_type`][determine_mime_type
plugin] plugin, which will make Shrine extract the MIME type from *file
content*.

```rb
Shrine.plugin :determine_mime_type
```
```rb
photo = Photo.create(image: StringIO.new("<?php ... ?>"))
photo.image.mime_type #=> "text/x-php"
```

By the default the UNIX [`file`] utility is used to determine the MIME type,
but you can also choose a different analyzer – see the plugin documentation for
more details.

### Other metadata

In addition to `size`, `filename`, and `mime_type`, you can also extract image
dimensions using the [`store_dimensions`][store_dimensions plugin] plugin, as
well as any custom metadata using the [`add_metadata`][add_metadata plugin]
plugin. Check out the [Extracting Metadata] guide for more details.

Note that you can also manually override extracted metadata by passing the
`:metadata` option to `Shrine#upload`:

```rb
uploaded_file = uploader.upload(file, metadata: { "filename" => "Matrix[1999].mp4", "foo" => "bar" })
uploaded_file.original_filename #=> "Matrix[1999].mp4"
uploaded_file.metadata["foo"]   #=> "bar"
```

## Processing

Shrine allows you to processing attached files either "on upload" or
"on-the-fly". For example, if your app is accepting image uploads, you can
generate a pre-defined set of of thumbnails as soon as the image is attached to
a record ("on upload"), or you can generate necessary thumbnails dynamically as
they're needed ("on-the-fly").

In both cases, for image processing you can use the **[ImageProcessing]** gem.
It provides a convenient unified API for processing with [ImageMagick] or
[libvips]. Here is an example of generating a thumbnail with ImageProcessing:

```sh
$ brew install imagemagick
```
```rb
# Gemfile
gem "image_processing", "~> 1.0"
```
```rb
require "image_processing/mini_magick"

thumbnail = ImageProcessing::MiniMagick
  .source(original_image)
  .resize_to_limit!(600, 400)

thumbnail #=> #<Tempfile:...> (thumbnail limited to 600x400)
```

### Processing on upload

Shrine allows you intercept when a cached file is being uploaded to permanent
storage, and perform any file processing you might want. The result of
processing can also be multiple files, such as thumbnails of various
dimensions. This processing can additionaly be delayed into a [background
job](#backgrounding).

The promotion hook is provided by the [`processing`][processing plugin] plugin,
while the ability to save multiple files is provided by the
[`versions`][versions plugin] plugin. Let's set up our uploader to generate
some thumbnails from the attached image:

```rb
require "image_processing/mini_magick"

class ImageUploader < Shrine
  plugin :processing # allows hooking into promoting
  plugin :versions   # enable Shrine to handle a hash of files
  plugin :delete_raw # delete processed files after uploading

  process(:store) do |io, context|
    versions = { original: io } # retain original

    io.download do |original|
      pipeline = ImageProcessing::MiniMagick.source(original)

      versions[:large]  = pipeline.resize_to_limit!(800, 800)
      versions[:medium] = pipeline.resize_to_limit!(500, 500)
      versions[:small]  = pipeline.resize_to_limit!(300, 300)
    end

    versions # return the hash of processed files
  end
end
```

After these files have been uploaded, their data will all be saved to the
`<attachment>_data` column. The attachment getter will then read them as a Hash
of [`Shrine::UploadedFile`][uploaded file] objects.

```rb
photo = Photo.create(image: image)

photo.image_data #=>
# '{
#   "original": {"id":"9sd84.jpg", "storage":"store", "metadata":{...}},
#   "large": {"id":"lg043.jpg", "storage":"store", "metadata":{...}},
#   "medium": {"id":"kd9fk.jpg", "storage":"store", "metadata":{...}},
#   "small": {"id":"932fl.jpg", "storage":"store", "metadata":{...}}
# }'

photo.image #=>
# {
#   :original => #<Shrine::UploadedFile @data={"id"=>"9sd84.jpg", ...}>,
#   :large    => #<Shrine::UploadedFile @data={"id"=>"lg043.jpg", ...}>,
#   :medium   => #<Shrine::UploadedFile @data={"id"=>"kd9fk.jpg", ...}>,
#   :small    => #<Shrine::UploadedFile @data={"id"=>"932fl.jpg", ...}>,
# }

photo.image[:medium]           #=> #<Shrine::UploadedFile>
photo.image[:medium].url       #=> "/uploads/store/lg043.jpg"
photo.image[:medium].size      #=> 5825949
photo.image[:medium].mime_type #=> "image/jpeg"
```

The `versions` plugin also expands `#<attachment>_url` to accept version names:

```rb
photo.image_url(:large) #=> "https://..."
```

For more details, see the [File Processing] guide.

### Processing on-the-fly

On-the-fly processing is provided by the
[`derivation_endpoint`][derivation_endpoint plugin] plugin. It provides a
mountable Rack app, which on request will call the processing code we defined.

We start by loading the plugin with a secret key and a path prefix to where
we'll mount the Rack app, and defining a "derivation" we want the app to call:

```rb
require "image_processing/mini_magick"

class ImageUploader < Shrine
  plugin :derivation_endpoint,
    secret_key: "<YOUR SECRET KEY>",
    prefix:     "derivations/image"

  derivation :thumbnail do |file, width, height|
    ImageProcessing::MiniMagick
      .source(file)
      .resize_to_limit!(width.to_i, height.to_i)
  end
end
```

Then we can mount the Rack app into our router (see [Mounting Endpoints] wiki
for other web frameworks):

```rb
# config/routes.rb (Rails)
Rails.application.routes.draw do
  mount ImageUploader.derivation_endpoint => "derivations/image"
end
```

Now we can generate URLs from attached files that on request will call the
processing we defined:

```rb
photo.image.derivation_url(:thumbnail, "600", "400")
#=> "/derivations/image/thumbnail/600/400/eyJpZCI6ImZvbyIsInN0b3JhZ2UiOiJzdG9yZSJ9?signature=..."
```

The `derivation_endpoint` plugin is highly customizable, for more details see
its [documentation][derivation_endpoint plugin].

## Context

The `#upload` (and `#delete`) methods accept a hash of options as the second
argument, which is forwarded down the chain and be available for processing,
extracting metadata and generating location.

```rb
uploader.upload(file, { foo: "bar" }) # context hash is forwarded to all tasks around upload
```

Some options are actually recognized by Shrine (such as `:location`,
`:upload_options`, and `:metadata`), some are added by plugins, and the rest are
there just to provide additional context, for more flexibility in performing
tasks and more descriptive logging.

The attacher automatically includes additional `context` information for each
upload and delete operation:

* `context[:record]` – model instance where the file is attached
* `context[:name]` – name of the attachment attribute on the model
* `context[:action]` – identifier for the action being performed (`:cache`, `:store`, `:recache`, `:backup`, ...)

```rb
class VideoUploader < Shrine
  process(:store) do |io, context|
    trim_video(io, 300) if context[:record].user.free_plan?
  end
end
```

## Validation

Shrine can perform file validations for files assigned to the model. The
validations are defined inside the `Attacher.validate` block, and you can load
the [`validation_helpers`][validation_helpers plugin] plugin to get convenient
file validation methods:

```rb
class DocumentUploader < Shrine
  plugin :validation_helpers

  Attacher.validate do
    validate_max_size 5*1024*1024, message: "is too large (max is 5 MB)"
    validate_mime_type_inclusion %w[application/pdf]
  end
end
```

```rb
user = User.new
user.cv = File.open("cv.pdf")
user.valid? #=> false
user.errors.to_hash #=> {:cv=>["is too large (max is 5 MB)"]}
```

For more details, see the [File Validation] guide and
[`validation_helpers`][validation_helpers plugin] plugin docs.

## Location

Before Shrine uploads a file, it generates a random location for it. By default
the hierarchy is flat – all files are stored in the root directory of the
storage. The [`pretty_location`][pretty_location plugin] plugin provides a nice
default hierarchy, but you can also override `#generate_location` with a custom
implementation:

```rb
class ImageUploader < Shrine
  def generate_location(io, context)
    type  = context[:record].class.name.downcase if context[:record]
    style = context[:version] == :original ? "originals" : "thumbs" if context[:version]
    name  = super # the default unique identifier

    [type, style, name].compact.join("/")
  end
end
```
```
uploads/
  photos/
    originals/
      la98lda74j3g.jpg
    thumbs/
      95kd8kafg80a.jpg
      ka8agiaf9gk4.jpg
```

Note that there should always be a random component in the location, so that
the ORM dirty tracking is detected properly. Inside `#generate_location` you
can also access the extracted metadata through `context[:metadata]`.

When uploading files, you can pass `:location` to bypass `#generate_location`:

```rb
uploader.upload(file, location: "some/specific/location.mp4")
```

## Direct uploads

To improve the user experience, it's recommended to upload files asynchronously
as soon they're selected. This way the UI is still responsive during upload, so
the user can fill in other fields while the files are being uploaded, and if
you display a progress bar they can see when the upload will finish.

These asynchronous uploads will have to go to an endpoint separate from the one
where the form is submitted. This can be an [endpoint in your app][simple
upload], or an [endpoint of a cloud service][presigned upload]. In either case,
the uploads should go to *temporary* storage (`:cache`), to ensure there won't
be any orphan files in the primary storage (`:store`).

Once files are uploaded on the client side, their data can be submitted to the
server and attached to a record, just like with raw files. The only difference
is that they won't be additionally uploaded to temporary storage on assignment,
as they were already uploaded on the client side.

```rb
# uploads the file to temporary storage and assigns it
photo.image = file

# assigns the already uploaded file
photo.image = '{"id":"...","storage":"cache","metadata":{...}}'
```

Note that by default **Shrine won't extract metadata from directly uploaded
files**, instead it will just copy metadata that was extacted on the client
side; see [this section][metadata direct uploads] for the rationale and
instructions on how to opt in.

For handling client side uploads it's recommended to use **[Uppy]**. Uppy is a
very flexible modern JavaScript file upload library, which happens to integrate
nicely with Shrine.

### Simple direct upload

The simplest approach is creating an upload endpoint in your app that will
receive uploads and forward them to the specified storage. You can use the
[`upload_endpoint`][upload_endpoint plugin] Shrine plugin to create a Rack app
that handles uploads, and mount it inside your application (see [Mounting
Endpoints] wiki for other web frameworks).

```rb
Shrine.plugin :upload_endpoint
```
```rb
# config/routes.rb (Rails)
Rails.application.routes.draw do
  # ...
  mount ImageUploader.upload_endpoint(:cache) => "/images/upload"
end
```

The above will add a `POST /images/upload` route to your app. You can now use
Uppy's [XHR Upload][uppy xhr-upload] plugin to upload selected files to this
endpoint, and have the uploaded file data submitted to your app. See [this
walkthrough][direct uploads walkthrough] for an example of adding simple direct
uploads from scratch.

### Presigned direct upload

If you want to free your app from receiving file uploads, you can also upload
files directly to the cloud (AWS S3, Google Cloud etc). In this flow the client
is required to first fetch upload parameters from the server, and then use these
parameters for the upload request.

You can use the [`presign_endpoint`][presign_endpoint plugin] Shrine plugin to
create a Rack app that generates these upload parameters (provided that the
underlying storage implements `#presign`), and mount it inside your application
(see [Mounting Endpoints] wiki for other web frameworks):

```rb
Shrine.plugin :presign_endpoint
```
```rb
# config/routes.rb (Rails)
Rails.application.routes.draw do
  # ...
  mount Shrine.presign_endpoint(:cache) => "/s3/params"
end
```

The above will add a `GET /s3/params` route to your app. You can now hook Uppy's
[AWS S3][uppy aws-s3] plugin to this endpoint and have it upload directly to
S3. See [this walkthrough][direct S3 uploads walkthrough] that shows adding
direct S3 uploads from scratch, as well as the [Direct Uploads to S3][direct S3
uploads guide] guide that provides some useful tips. Also check out the
[Roda][roda demo] / [Rails][rails demo] demo app which implements multiple
uploads directly to S3.

### Resumable direct upload

If your app is dealing with large uploads (e.g. videos), keep in mind that it
can be challening for your users to upload these large files to your app. Many
users might not have a great internet connection, and if it happens to break at
any point during uploading, they need to retry the upload from the beginning.

This problem has been solved by **[tus]**. tus is an open protocol for
resumable file uploads, which enables the client and the server to achieve
reliable file uploads even on unstable connections, by enabling the upload to
be resumed in case of interruptions, even after the browser was closed or the
device was shut down.

[tus-ruby-server] provides a Ruby server implementation of the tus protocol.
Uppy's [Tus][uppy tus] plugin can then be configured to do resumable uploads to
a tus-ruby-server instance, and then the uploaded files can be attached to the
record with the help of [shrine-tus]. See [this walkthrough][resumable uploads
walkthrough] that adds resumable uploads from scratch, as well as the
[demo][resumable demo] for a complete example.

```rb
# Gemfile
gem "tus-server", "~> 2.0"
```
```rb
# config/routes.rb (Rails)
Rails.application.routes.draw do
  # ...
  mount Tus::Server => "/files"
end
```

Alternatively, you can have resumable uploads directly to S3 using Uppy's [AWS
S3 Multipart][uppy aws-s3-multipart] plugin, accompanied with the
[uppy-s3_multipart] gem.

```rb
# Gemfile
gem "uppy-s3_multipart", "~> 0.3"
```
```rb
# config/initializers/shrine.rb
# ...
Shrine.plugin :uppy_s3_multipart
```
```rb
# config/routes.rb (Rails)
Rails.application.routes.draw do
  # ...
  mount Shrine.uppy_s3_multipart(:cache) => "/s3/multipart"
end
```

## Backgrounding

Shrine is the first file attachment library designed for backgrounding support.
Moving phases of managing file attachments to background jobs is essential for
scaling and good user experience, and Shrine provides a
[`backgrounding`][backgrounding plugin] plugin which makes it easy to plug in
your favourite backgrounding library:

```rb
Shrine.plugin :backgrounding
Shrine::Attacher.promote { |data| PromoteJob.perform_async(data) }
Shrine::Attacher.delete { |data| DeleteJob.perform_async(data) }
```
```rb
class PromoteJob
  include Sidekiq::Worker
  def perform(data)
    Shrine::Attacher.promote(data)
  end
end
```
```rb
class DeleteJob
  include Sidekiq::Worker
  def perform(data)
    Shrine::Attacher.delete(data)
  end
end
```

The above puts promoting (uploading cached file to permanent storage) and
deleting of files for all uploaders into background jobs using Sidekiq. You can
also use [any other backgrounding library][backgrounding libraries].

## Clearing cache

Shrine doesn't automatically delete files uploaded to temporary storage, instead
you should set up a separate recurring task that will automatically delete old
cached files.

Most of Shrine storage classes come with a `#clear!` method, which you can call
in a recurring script. For FileSystem and S3 storage it would look like this:

```rb
# FileSystem storage
file_system = Shrine.storages[:cache]
file_system.clear!(older_than: Time.now - 7*24*60*60) # delete files older than 1 week
```
```rb
# S3 storage
s3 = Shrine.storages[:cache]
s3.clear! { |object| object.last_modified < Time.now - 7*24*60*60 } # delete files older than 1 week
```

Note that for AWS S3 you can also configure bucket lifecycle rules to do this
for you. This can be done either from the [AWS Console][S3 lifecycle console]
or via an [API call][S3 lifecycle API]:

```rb
require "aws-sdk-s3"

client = Aws::S3::Client.new(
  access_key_id:     "<YOUR KEY>",
  secret_access_key: "<YOUR SECRET>",
  region:            "<REGION>",
)

client.put_bucket_lifecycle_configuration(
  bucket: "<YOUR BUCKET>",
  lifecycle_configuration: {
    rules: [{
      expiration: { days: 7 },
      filter: { prefix: "cache/" },
      id: "cache-clear",
      status: "Enabled"
    }]
  }
)
```

## Logging

Shrine ships with the `logging` which automatically logs processing, uploading,
and deleting of files. This can be very helpful for debugging and performance
monitoring.

```rb
Shrine.plugin :logging
```
```
2015-10-09T20:06:06.676Z #25602: STORE[cache] ImageUploader[:avatar] User[29543] 1 file (0.1s)
2015-10-09T20:06:06.854Z #25602: PROCESS[store]: ImageUploader[:avatar] User[29543] 1-3 files (0.22s)
2015-10-09T20:06:07.133Z #25602: DELETE[destroyed]: ImageUploader[:avatar] User[29543] 3 files (0.07s)
```

## Settings

Each uploader can store generic settings in the `opts` hash, which can be
accessed in other uploader actions. You can store there anything that you find
convenient.

```rb
Shrine.opts[:type] = "file"

class DocumentUploader < Shrine; end
class ImageUploader < Shrine
  opts[:type] = "image"
end

DocumentUploader.opts[:type] #=> "file"
ImageUploader.opts[:type]    #=> "image"
```

Because `opts` is cloned in subclasses, overriding settings works with
inheritance. The `opts` hash is used internally by plugins to store
configuration.

## Inspiration

Shrine was heavily inspired by [Refile] and [Roda]. From Refile it borrows the
idea of "backends" (here named "storages"), attachment interface, and direct
uploads. From Roda it borrows the implementation of an extensible plugin
system.

## Similar libraries

* Paperclip
* CarrierWave
* Dragonfly
* Refile
* Active Storage

## Code of Conduct

Everyone interacting in the Shrine project’s codebases, issue trackers, and
mailing lists is expected to follow the [Shrine code of conduct][CoC].

## License

The gem is available as open source under the terms of the [MIT License].

[Advantages of Shrine]: /doc/advantages.md#readme
[backgrounding libraries]: https://github.com/shrinerb/shrine/wiki/Backgrounding-Libraries
[Creating Plugins]: /doc/creating_plugins.md#readme
[Creating Storages]: /doc/creating_storages.md#readme
[Extracting Metadata]: /doc/metadata.md#readme
[File Processing]: /doc/processing.md#readme
[File Validation]: /doc/validation.md#readme
[Mounting Endpoints]: https://github.com/shrinerb/shrine/wiki/Mounting-Endpoints
[Retrieving Uploads]: /doc/retrieving_uploads.md#readme
[Using Attacher]: /doc/attacher.md#readme
[`Shrine::UploadedFile`]: https://shrinerb.com/rdoc/classes/Shrine/UploadedFile/InstanceMethods.html

[attacher]: #attacher
[backgrounding]: #backgrounding
[context]: #context
[direct uploads]: #direct-uploads
[io abstraction]: #io-abstraction
[location]: #location
[metadata]: #metadata
[on upload]: #processing-on-upload
[on-the-fly]: #processing-on-the-fly
[plugin system]: #plugin-system
[presigned upload]: #presigned-direct-upload
[resumable upload]: #resumable-direct-upload
[simple upload]: #simple-direct-upload
[storage]: #storage
[uploaded file]: #uploaded-file
[uploader]: #uploader
[validation]: #validation

[Cloudinary]: https://github.com/shrinerb/shrine-cloudinary
[FileSystem]: /doc/storage/file_system.md#readme
[GCS]: https://github.com/renchap/shrine-google_cloud_storage
[S3 lifecycle API]: https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/S3/Client.html#put_bucket_lifecycle_configuration-instance_method
[S3 lifecycle Console]: http://docs.aws.amazon.com/AmazonS3/latest/UG/lifecycle-configuration-bucket-no-versioning.html
[S3]: /doc/storage/s3.md#readme
[AWS S3]: https://aws.amazon.com/s3/
[MinIO]: https://min.io/
[DigitalOcean Spaces]: https://www.digitalocean.com/products/spaces/
[Transloadit]: https://github.com/shrinerb/shrine-transloadit

[ImageMagick]: https://imagemagick.org/
[ImageProcessing::MiniMagick]: https://github.com/janko/image_processing/blob/master/doc/minimagick.md#readme
[ImageProcessing::Vips]: https://github.com/janko/image_processing/blob/master/doc/vips.md#readme
[ImageProcessing]: https://github.com/janko/image_processing
[`file`]: http://linux.die.net/man/1/file
[libvips]: http://libvips.github.io/libvips/

[Uppy]: https://uppy.io
[direct S3 uploads guide]: /doc/direct_s3.md#readme
[direct S3 uploads walkthrough]: https://github.com/shrinerb/shrine/wiki/Adding-Direct-S3-Uploads
[direct uploads walkthrough]: https://github.com/shrinerb/shrine/wiki/Adding-Direct-App-Uploads
[rails demo]: https://github.com/erikdahlstrand/shrine-rails-example
[resumable demo]: https://github.com/shrinerb/shrine-tus-demo
[resumable uploads walkthrough]: https://github.com/shrinerb/shrine/wiki/Adding-Resumable-Uploads
[roda demo]: https://github.com/shrinerb/shrine/tree/master/demo
[shrine-tus]: https://github.com/shrinerb/shrine-tus
[tus-ruby-server]: https://github.com/janko/tus-ruby-server
[tus]: https://tus.io
[uppy aws-s3-multipart]: https://uppy.io/docs/aws-s3-multipart/
[uppy aws-s3]: https://uppy.io/docs/aws-s3/
[uppy tus]: https://uppy.io/docs/tus/
[uppy xhr-upload]: https://uppy.io/docs/xhr-upload/
[uppy-s3_multipart]: https://github.com/janko/uppy-s3_multipart
[metadata direct uploads]: /doc/metadata.md#direct-uploads

[activerecord plugin]: /doc/plugins/activerecord.md#readme
[add_metadata plugin]: /doc/plugins/add_metadata.md#readme
[backgrounding plugin]: /doc/plugins/backgrounding.md#readme
[derivation_endpoint plugin]: /doc/plugins/derivation_endpoint.md#readme
[determine_mime_type plugin]: /doc/plugins/determine_mime_type.md#readme
[hanami plugin]: https://github.com/katafrakt/hanami-shrine
[mongoid plugin]: https://github.com/shrinerb/shrine-mongoid
[presign_endpoint plugin]: /doc/plugins/presign_endpoint.md#readme
[pretty_location plugin]: /doc/plugins/pretty_location.md#readme
[processing plugin]: /doc/plugins/processing.md#readme
[rack_file plugin]: /doc/plugins/rack_file.md#readme
[sequel plugin]: /doc/plugins/sequel.md#readme
[store_dimensions plugin]: /doc/plugins/store_dimensions.md#readme
[upload_endpoint plugin]: /doc/plugins/upload_endpoint.md#readme
[validation_helpers plugin]: /doc/plugins/validation_helpers.md#readme
[versions plugin]: /doc/plugins/versions.md#readme

[Shrine]: https://shrinerb.com
[external]: https://shrinerb.com/#external
[plugins]: https://shrinerb.com/#plugins
[CoC]: CODE_OF_CONDUCT.md
[MIT License]: http://opensource.org/licenses/MIT
[Refile]: https://github.com/refile/refile
[Roda]: https://github.com/jeremyevans/roda
