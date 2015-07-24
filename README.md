![General Assembly Logo](http://i.imgur.com/ke8USTq.png)

# Image Upload with the Paperclip Gem

## Objectives

By the end of this lesson, students should be able to:

- Use Paperclip to implemenet image upload on localhost.
- Integrate Paperclip with S3.

## Prerequisites

- Ruby
- Rails MVC
- Postgres

## Paperclip

Paperclip is a gem that was created by a development shop called _**thoughtbot**_; it's a tool to simplify the process of integrating image upload (a common feature for web applications) into Rails. Although Paperclip can be used in conjunction with 3rd party storage systems like Amazon S3, it can also be used simply to upload images to an app on localhost.

Together, we're going to walk through the steps of how you take an ordinary Rails application and incorporate image upload. Afterwards, if there's time, we'll look at how to integrate S3 as well.

### Paperclip with Local Storage

Like many things in Rails, incorporating Paperclip into a project involves following a recipe. Code along if you can!
In this example, we'll be building a simple CRUD app for movies, and adding image upload in order to imbue each of these `Movie` resources with a 'poster' image.

1. First, install ImageMagick from HomeBrew - Paperclip uses ImageMagick to process the images it receives.

  ```bash
    brew install ImageMagick
  ```

2. Make your new Rails app.

  a. `rails new demo_movie_app -T --database=postgresql`

  b. `bundle install`

  c. `rake db:create`

  d. `rails g migration CreateMovies title:string director:string year:integer`

  e. Make a model (Movie), and populate `seeds.rb`

  f. `rake db:migrate`

  g. Make routes and a controller to respond to them. Leave off the `new` and `edit` controller actions for now. Don't forget to set up strong params inside the controller!

  h. Test the app - does it work? You might need to add things related to CORS.

3. Once you've made your Rails app and have confirmed that it's working correctly, you can move onto the next step - adding attachments to models.

  a. Add the `paperclip` gem to your Gemfile. Afterwards, run `bundle install`.

    ```ruby
    gem "paperclip", :git => "git://github.com/thoughtbot/paperclip.git"
    ```

  b. Edit the model that you want to add the attachment to, as follows:

    ```ruby
    has_attached_file :poster,  #Or whatever you want to call the image you're uploading.
                :styles => { :medium => "300x300>", :thumb => "100x100>" },
                :default_url => "/images/:style/missing.png"
    validates_attachment_content_type :poster, :content_type => /\Aimage\/.*\Z/
    ```

  c. Create a new migration, either using a generator (`rails g paperclip movie poster`) or by writing out your migration file by hand. Once your migration file is ready, run `rake db:migrate` to run your migration.

  d. Create two files inside the `views` folder, `edit.html.erb` and `new.html.erb`; the contents of these files should be more or less the same. These two files will be used to generate forms, which we need in order to upload images.

    ```erb
    <%= form_for @movie, :html => { :multipart => true } do |form| %>
      <%= form.label :title%>
      <%= form.text_field :title%>
      <%= form.label :director%>
      <%= form.text_field :director%>
      <%= form.label :year%>
      <%= form.text_field :year%>
      <%= form.label 'Upload Poster'%>
      <%= form.file_field :poster%>
      <%= form.submit 'Update Movie' %>
      <%= link_to 'Back', movies_path, class: 'button' %>
    <% end %>
    ```
    > Be sure to replace the fields and labels to be in keeping with the properties of your model.

  e. Edit your controller. Give your controller two new methods to line up with the views you've just created (edit and new); make sure that you **don't** have these actions render JSON. Then, adjust your strong params function to permit the new property (in this case, `:poster`). Finally, add a method to delete

  f. Once all of this is done, restart Rails and try visiting your routes.

  Once you upload an image, it will (by default) get added to a directory within the `public` directory in your Rails app. To access these images, you can either create a Rails View to show them, or (more likely, in the case of dealing with a JSON API), you can send back a link to where the file is available within the app.

#### Your Turn :: Paperclip

You're going to work in pairs to build an CRUD app for songs. Follow the recipe above, and do your best!

### Paperclip with S3
Our locally-hosted application is, as you've just seen, fully capable of accepting file uploads and managing them. So when we host to Heroku, that should work too, right?

Wrong.

Although Heroku's system will swear up and down that it can handle image upload like this, and give you signals that it's doing the right thing, _it isn't_. So when we deploy our apps, we need to find an alternative storage solution.

One of the more common solutions is Amazon's Simple Storage Service (S3). With only a few modifications, we can configure our app so that it's always storing things to Amazon instead of storing them locally.

Ready to take the leap? Let's do it!

1. If you don't have one yet, sign up for an Amazon Web Services account. (Incidentally, GA has a [pretty sweet promotional deal](images/GA-alumni-perks.png) that you can use once you create an account - [click here](http://aws.amazon.com/activate/event/20gafr15/) to claim it.)

2. Inside AWS, create a new IAM user, and write down (a) the AWS access key ID and (b) the AWS secret access key. You'll need this information later, and if you don't record it now, you'll have to re-generate it. Once they's done, create a Resource Group with admin privileges, and add the new user to it.

3. Then, create a new 'Bucket', and write down the name of that bucket.

4. Add the following gems to your Gemfile and run `bundle install`.

  ```ruby
  gem 'aws-sdk-v1'
  gem 'dotenv-rails'
  ```
5. Create an empty `.env` file in the root of your Rails app, and add `.env` to your `.gitignore`.

  > If you don't have a `.gitignore`, **you need to make one**. The `.gitignore` file is all that keeps us from commiting sensitive information to your repo for all time.

6. Add the following things to your new `.env` file.

  ```bash
  S3_BUCKET_NAME=...
  AWS_ACCESS_KEY_ID=...
  AWS_SECRET_ACCESS_KEY=...
  ```

  Plug in the values that you got when setting up AWS, and delete them from wherever else you've written them down.

7. Add the following code to just before the final `end` in both the `config/environments/production.rb` and `config/environments/development.rb` files.

  ```ruby
  config.paperclip_defaults = {
    :storage => :s3,
    s3_credentials: {
      bucket: ENV['S3_BUCKET_NAME'],
      access_key_id: ENV['AWS_ACCESS_KEY_ID'],
      secret_access_key: ENV['AWS_SECRET_ACCESS_KEY']
    }
  }
  ```
  Assuming you've set up your `.env` file correctly, and have installed the `dotenv-rails` gem, these references should point to the variables in `.env`, _even as they remain invisible to Git_.

  > "But wait! If Git can't see our important secrets, how will they transfer over when we deploy?"
  Great question. The precise way of handling secret information across platforms depends on how you've decided to manage secret information within your app; assuming we stick with the system we've been using, the way that we'd share secret information with a hosting service like Heroku is by establishing a secure, private line of communication between us and the host, and transferring the secrets through that protected channel. We'll look at this more next week when we start deploying to Heroku.

#### Your Turn :: Paperclip with S3

In the same pairs as before, convert over your Rails app to link up with S3 instead of storing images on localhost. If you can get it working, trying creating a simple front-end app to plug into this new back-end!

## Additional References
- [Paperclip's GitHub Page](https://github.com/thoughtbot/paperclip)
- [Heroku's Paperclip + S3 Walkthrough](https://devcenter.heroku.com/articles/paperclip-s3)
- [RubyDocs Page on Paperclip](http://www.rubydoc.info/gems/paperclip/Paperclip)
- [RubyDocs Page on using S3 with Paperclip](http://www.rubydoc.info/gems/paperclip/Paperclip/Storage/S3)
