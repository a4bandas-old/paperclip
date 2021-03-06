Paperclip-Cloudfiles
=================================

Paperclip-Cloudfiles is intended as an easy file attachment library for ActiveRecord and all attachments are served from Rackspace's Cloudfiles.

Some features include:

* Files aren't saved to their final locations on disk, nor are they deleted if set to nil, until
ActiveRecord::Base#save is called.
* Validations are managed based on size and
presence, if required.
* It can transform its assigned image into thumbnails if
needed, by installing ImageMagick (which,
for most modern Unix-based systems, is as easy as installing the right
packages).
* Attached files are either saved to the filesystem, or to Rackspace Cloudfiles, and referenced in the
browser by an easily understandable specification, which has sensible and
useful defaults.

Note: The Thoughtbot guys have indicated that they
don't want to pull any code into the official Paperclip mainline that they don't 
personally use on projects, so until they discover the joy of Cloud Files, this 
fork is available on RubyGems.org at http://rubygems.org/gems/paperclip-cloudfiles

The complete [RDoc](http://rdoc.info/github/minter/paperclip) is online.

Changes in this repo
------------

Allowed to refresh images of classes with namespaces. For example:

    rake paperclip:refresh CLASS='User::Asset'

Requirements
------------

ImageMagick must be installed and Paperclip must have access to it. Run `which convert` (one of the ImageMagick
utilities).  It might return `/usr/local/bin/convert` or `/usr/bin/convert`.

Add the returned line to `config/environments/development.rb)` and to `config/environments/production.rb)`:

    Paperclip.options[:command_path] = "/usr/local/bin/"

Installation
------------

Include the gem in your Gemfile (Rails 3 or Rails 2.x with Bundler):
  
    gem 'cloudfiles'
    gem 'cocaine' #a dependency that paperclilp didn't pick up yet
    gem 'paperclip-cloudfiles', :require => 'paperclip'

 (Rails 2 only) In your environment.rb:

    config.gem "paperclip-cloudfiles", :lib => 'paperclip'

This is because the gem name and the library name don't match.

Quick Start
-----------

Create `config/rackspace_cloudfiles.yml`

    
    DEFAULTS: &DEFAULTS
      username: yourusernamehere
      api_key: yourapikeyhere

    development:
      <<: *DEFAULTS
      container: dev_avatars

    test:
      <<: *DEFAULTS
      container: test_avatars

    production:
      <<: *DEFAULTS
      container: avatars


Declare that your model has an attachment with the has_attached_file method, and give it a name.
In your model:

```ruby
    class User < ActiveRecord::Base
      
      # More information about the has_attached_file options are available in the
      # documentation of Paperclip::ClassMethods.
      
      has_attached_file :avatar,
        :styles => { :medium => "300x300>",  :thumb => "100x100>" },
        :storage => :cloud_files,
        :cloudfiles_credentials => "#{Rails.root}/config/rackspace_cloudfiles.yml"

      # Validation Methods:
	
      validates_attachment_presence :avatar
      validates_attachment_content_type :avatar, :content_type => ['image/jpeg', 'image/png']
      validates_attachment_size :avatar, :in => 1..1.megabyte
    end
```
    
Paperclip will wrap up up to four attributes (all prefixed with that attachment's name,
so you can have multiple attachments per model if you wish) and give them a
friendly front end.

In your migrations:

```ruby
class AddAvatarColumnsToUser < ActiveRecord::Migration

  def self.up
      add_column :users, :avatar_file_name,    :string
      add_column :users, :avatar_content_type, :string
      add_column :users, :avatar_file_size,    :integer
      add_column :users, :avatar_updated_at,   :datetime
  end

  def self.down
      remove_column :users, :avatar_file_name
      remove_column :users, :avatar_content_type
      remove_column :users, :avatar_file_size
      remove_column :users, :avatar_updated_at
  end
end
```

In your edit and new views:

    <% form_for :user, @user, :url => user_path, :html => { :multipart => true } do |form| %>
      <%= form.file_field :avatar %>
    <% end %>

In your controller:

    def create
      @user = User.create( params[:user] )
    end

In your show view:

    <%= image_tag @user.avatar.url %>
    <%= image_tag @user.avatar.url(:medium) %>
    <%= image_tag @user.avatar.url(:thumb) %>
 

Storage
-------

The files that are assigned as attachments are, by default, placed in the
directory specified by the :path option to has_attached_file. By default, this
location is ":rails_root/public/system/:attachment/:id/:style/:filename". This
location was chosen because on standard Capistrano deployments, the
public/system directory is symlinked to the app's shared directory, meaning it
will survive between deployments. For example, using that :path, you may have a
file at

    /data/myapp/releases/20081229172410/public/system/avatars/13/small/my_pic.png

_NOTE: This is a change from previous versions of Paperclip, but is overall a
safer choice for the default file store._

You may also choose to store your files using Rackspace's Cloud Files service. You can find more information about Cloud Files storage at the description for Paperclip::Storage::CloudFile

Note:

Files on the local filesystem (and in the Rails app's public directory), and on Rackspace Cloudfiles, will be available to the internet at large. For the filesystem, if you require access control, it's
possible to place your files in a different location. You will need to change
both the :path and :url options in order to make sure the files are unavailable
to the public. Both :path and :url allow the same set of interpolated
variables.

Post Processing
---------------

Paperclip supports an extensible selection of post-processors. When you define
a set of styles for an attachment, by default it is expected that those
"styles" are actually "thumbnails". However, you can do much more than just
thumbnail images. By defining a subclass of Paperclip::Processor, you can
perform any processing you want on the files that are attached. Any file in
your Rails app's lib/paperclip_processors directory is automatically loaded by
paperclip, allowing you to easily define custom processors. You can specify a
processor with the :processors option to has_attached_file:

    has_attached_file :scan, :styles => { :text => { :quality => :better } },
                             :processors => [:ocr]

This would load the hypothetical class Paperclip::Ocr, which would have the
hash "{ :quality => :better }" passed to it along with the uploaded file. For
more information about defining processors, see Paperclip::Processor.

The default processor is Paperclip::Thumbnail. For backwards compatability
reasons, you can pass a single geometry string or an array containing a
geometry and a format, which the file will be converted to, like so:

    has_attached_file :avatar, :styles => { :thumb => ["32x32#", :png] }

This will convert the "thumb" style to a 32x32 square in png format, regardless
of what was uploaded. If the format is not specified, it is kept the same (i.e.
jpgs will remain jpgs).

Multiple processors can be specified, and they will be invoked in the order
they are defined in the :processors array. Each successive processor will
be given the result of the previous processor's execution. All processors will
receive the same parameters, which are what you define in the :styles hash.
For example, assuming we had this definition:

    has_attached_file :scan, :styles => { :text => { :quality => :better } },
                             :processors => [:rotator, :ocr]

then both the :rotator processor and the :ocr processor would receive the
options "{ :quality => :better }". This parameter may not mean anything to one
or more or the processors, and they are expected to ignore it.

_NOTE: Because processors operate by turning the original attachment into the
styles, no processors will be run if there are no styles defined._

Events
------

Before and after the Post Processing step, Paperclip calls back to the model
with a few callbacks, allowing the model to change or cancel the processing
step. The callbacks are `before_post_process` and `after_post_process` (which
are called before and after the processing of each attachment), and the
attachment-specific `before_<attachment>_post_process` and
`after_<attachment>_post_process`. The callbacks are intended to be as close to
normal ActiveRecord callbacks as possible, so if you return false (specifically
- returning nil is not the same) in a before_ filter, the post processing step
will halt. Returning false in an after_ filter will not halt anything, but you
can access the model and the attachment if necessary.

_NOTE: Post processing will not even *start* if the attachment is not valid
according to the validations. Your callbacks and processors will *only* be
called with valid attachments._

Testing
-------

Paperclip provides rspec-compatible matchers for testing attachments. See the
documentation on [Paperclip::Shoulda::Matchers](http://rubydoc.info/gems/paperclip/Paperclip/Shoulda/Matchers)
for more information.

Contributing
------------

If you'd like to contribute a feature or bugfix: Thanks! To make sure your
fix/feature has a high chance of being included, please read the following
guidelines:

1. Ask on the mailing list[http://groups.google.com/group/paperclip-plugin], or 
   post a new GitHub Issue[http://github.com/thoughtbot/paperclip/issues].
2. Make sure there are tests! We will not accept any patch that is not tested.
   It's a rare time when explicit tests aren't needed. If you have questions
   about writing tests for paperclip, please ask the mailing list.

Credits
-------

![thoughtbot](http://thoughtbot.com/images/tm/logo.png)

Paperclip is maintained and funded by [thoughtbot, inc](http://thoughtbot.com/community)

Thank you to all [the contributors](https://github.com/thoughtbot/paperclip/contributors)!

The names and logos for thoughtbot are trademarks of thoughtbot, inc.

License
-------

Paperclip is Copyright © 2008-2011 thoughtbot. It is free software, and may be redistributed under the terms specified in the MIT-LICENSE file.
