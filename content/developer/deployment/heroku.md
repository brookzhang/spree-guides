---
title: "Deploying to Heroku"
section: deployment
---

## Overview

This article will walk you through configuring and deploying your Spree
application to Heroku.

This guide assumes that your application is deploy-ready and that you have a
Heroku application already created on the Heroku stack for this application. If
you don't have a Heroku app already, follow [this
guide](https://devcenter.heroku.com/articles/creating-apps).

***
Heroku's tools assume that your application is version controlled by Git, as
Git is used to push the code to Heroku.
***

## Configuring your application

There are three major things you will need to edit before you can deploy your
application to Heroku: your Gemfile to specify Ruby 1.9.3, the asset pipeline's `initialize_on_precompile` setting
and S3 settings.

### Using Ruby 1.9.3

***
If you are not using Spree 2.0.0 and above, then you can safely ignore this section.
***

Spree 2.0.0 requires a version of Ruby greater than or equal to Ruby 1.9.3. By default, Heroku uses Ruby 1.9.2, which *will not work* with Spree. To force Heroku to use Ruby 1.9.3, put this line in your Gemfile:

```ruby
ruby '1.9.3'```

### Asset Pipeline

When deploying to Heroku by default Rails will attempt to intialize itself
before the assets are precompiled. This step will fail because the application
will attempt to establish a database connection, which Heroku will not have set
up yet.

To work around this issue, put this line underneath the other `config.assets`
lines inside `config/application.rb`:

```ruby
config.assets.initialize_on_precompile = false
```

The assets for your application will still be precompiled, it's just that Rails
won't be intialized during this process.

### S3 Support

Because Heroku's filesystem is readonly, you will need to configure Spree to
upload the assets to an off-site server, such as S3. If you don't have an S3
account already, you can [set one up here](http://aws.amazon.com/s3/)

This guide will assume that you have an S3 account already, along with a bucket
under that account for your files to go into, and that you have generated the
access key and secret for your S3 account.

To configure Spree to upload images to S3, put these lines into
`config/initializers/spree.rb`:

```ruby
Spree.config do |config|
  config.use_s3 = true
  config.s3_bucket = '<bucket>'
  config.s3_access_key = "<key>"
  config.s3_secret = "<secret>"
end
```

If you're using the Western Europe S3 server, you will need to set two
additional options inside this block:

```ruby
Spree.config do |config|
  ...
  config.attachment_url = ":s3_eu_url"
  config.s3_host_alias = "s3-eu-west-1.amazonaws.com"
end
```

And additionally you will need to tell paperclip how to construct the URLs for
your images by placing this code outside the +config+ block inside
`config/initializers/spree.rb`:

```ruby
Paperclip.interpolates(:s3_eu_url) do |attachment, style|
"#{attachment.s3_protocol}://#{Spree::Config[:s3_host_alias]}/#{attachment.bucket_name}/#{attachment.path(style).gsub(%r{^/},"")}"
end
```

## Pushing to Heroku

Once you have configured the above settings, you can push your Spree application
to Heroku:

```bash
$ git push heroku master
```

Once your application is on Heroku, you will need to set up the schema by
running this command:

```bash
heroku run rake db:migrate```

You may then wish to set up an admin user as well which can be done by loading
the rails console:

```bash
heroku run rails console```

And then running this code:

```ruby
user = Spree::User.create!(:email => "you@example.com", :password => "yourpassword")
user.spree_roles.create!(:name => "admin")```

Exit out of the console and then attempt to sign in to your application to
verify these credentials.

## SSL Support

For information about SSL support with Heroku, please read their [SSL
Guide](https://devcenter.heroku.com/articles/ssl)
