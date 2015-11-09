![General Assembly Logo](http://i.imgur.com/ke8USTq.png)

# Image Upload with the Paperclip Gem

## Objectives

By the end of this lesson, students should be able to:

- Use Paperclip to implement image upload on localhost.
- Integrate Paperclip with S3.

## Prerequisites

- Rails MVC
- Postgres
- jQuery

## Paperclip

Paperclip is a gem that was created by a dev shop called [**thoughtbot**](https://thoughtbot.com/); it's a tool to simplify the process of integrating image upload (a common feature for web applications) into Rails. Although Paperclip can be used in conjunction with 3rd party storage systems like Amazon S3, it can also be used simply to upload images to an app on localhost.

Together, we're going to walk through the steps of how you take an ordinary Rails application and incorporate image upload. Afterwards, if there's time, we'll look at how to integrate S3 as well.

### Paperclip with Local Filesystem Storage

Like many things in Rails, incorporating Paperclip into a project involves following a recipe. Code along if you can!
In this example, we'll be building a simple CRUD app for movies, and adding image upload in order to imbue each of these `Movie` resources with a 'poster' image.

1. First, install ImageMagick using HomeBrew - Paperclip uses ImageMagick to process the images it receives.

  ```bash
    brew install ImageMagick
  ```

2. Make your new Rails app.
  > If you already have a Rails application ready, and just want to add Paperclip to it, you can skip this step.

  a. `rails new mini_movie_app -T --database=postgresql`

  b. `bundle install`

  c. `bundle exec rake db:create`

  d. `rails g model Movie title release_year:integer`

  e. `bundle exec rake db:migrate`

  f. Write a Ruby script to populate the database (e.g. `scripts/populate_movies.rb`) with some data.

  g. Make some routes and a controller to respond to them. Don't forget to set up strong params inside the controller!

  ```ruby
  # config/routes.rb
  Rails.application.routes.draw do
    resources :movies, only:[:index, :show, :create]
  end
  ```

  ```ruby
  # app/controllers/movies_controller.rb
  class MoviesController < ApplicationController

    def index
      render json: Movie.all
    end

    def show
      movie = Movie.find(params[:id])
      render json: movie
    end

    def create
      movie = Movie.new(movie_params)
      if movie.save
        render json: movie
      else
        render json: movie.errors, status: :unprocessable_entity
      end
    end

    private
    def movie_params
      params.require(:movie).permit(:title, :release_year)
    end

  end
  ```

  h. Test the API with Postman/`curl` - does it work?
  > Don't forget to set up a CORS policy so that your API is accessible from a separate front-end!

3. Once you've made your Rails app and have confirmed that it's working correctly, you can move onto the next step - adding Paperclip attachments to models.

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

    # By default, every file uploaded will be named 'data'.
    # This renames the file to a timestamp, preventing name collisions.
    # This will also fix browser caching issues for updates
    def rename_poster
      self.poster.instance_write :file_name, "#{Time.now.to_i.to_s}.png"
    end

    before_post_process :rename_poster
    ```

  c. Create a new migration, either using a generator (`rails g paperclip movie poster`) or by writing out your migration file by hand. Once your migration file is ready, run `rake db:migrate` to run your migration.

  d. Edit the 'strong params' function inside your controller to permit the new 'poster' property.

  ```ruby
  def movie_params
    params.require(:movie).permit(:title, :release_year, :poster)
  end
  ```

  e. Create your front-end app. Although the Paperclip gem provides a relatively simple way to implement image upload into an app, sending an image from the client to the API is tricky - you need to make use of JavaScript's `FileReader` prototype in order to make things work. An example implementation is below (credit belongs to [Max Blaushild](https://github.com/MaxBlaushild))

  ```javascript
  // main.js
  $(document).ready(function(){

    $('#submit').on('click', function(e){
      e.preventDefault();
      // creates a new instance of the FileReader object prototype
      var reader = new FileReader();

      //setting a function to be executed every time the reader successfully completes a read operation
      reader.onload = function(event){
        // once the data url has been loaded, make the ajax request with the result set as the value to key 'poster'
        $.ajax({
          url: 'http://localhost:3000/movies',
          method: 'POST',
          data: { movie: {
            title: $('#title').val(),
            director: $('#director').val(),
            poster: event.target.result
          } }
        }).done(function(response){

        }).fail(function(response){
          console.error('Whoops!');
        })
      };

      // read the first file of the file input
      $fileInput = $('#poster');
      reader.readAsDataURL($fileInput[0].files[0]);

    });
  });
  ```

  Here is the HTML page that this JavaScript code refers to.

  ```html
  /* index.html */
  <!DOCTYPE html>
  <html>
    <head>
      <title>Paperclip Example Front-End</title>
    </head>
    <body>
      <h1> Using Paperclip Over a JSON API </h1>
      <form>
        <label> Title
          <input type="text" id="title"/>
        </label>
        <label> Director
          <input type="text" id="director"/>
        </label>
        <label> Poster
          <input type="file" id="poster"/>
        </label>
        <button id="submit">Create Movie</button>
      </form>
      <script src="https://code.jquery.com/jquery-2.1.4.js"></script>
      <script src="main.js"></script>
    </body>
  </html>
  ```

  e. Once all of this is done, restart the Rails server and try to upload an image from your front-end.

  Once you upload an image, it will (by default) get added to a directory within the `public` directory in your Rails app. To give the front-end access to this image, you can send back a link to where the file is available within the app.

#### Your Turn :: Paperclip

In your groups, add Paperclip to the IMDB clone from the second summative lesson - make it so that you can attach Posters to Movies, or Photos to People.

### Paperclip with S3 Storage
Our locally-hosted application is, as you've just seen, fully capable of accepting file uploads and managing them. So when we deploy to Heroku, that should work too, right?

Wrong.

As was explained in the previous lesson, Heroku provides an ephemeral filesystem, so any files that we try to create or add to Heroku while it's running will _not be added_. If we want image upload to work when we deploy our apps, we need to find an alternative storage solution.

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

  > If you don't have a `.gitignore`, **you need to make one**. The `.gitignore` file is all that keeps us from committing sensitive information to your repo for all time.

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
  Great question. The precise approach we use for handling secret information across platforms depends on how secret information is managed within your app; assuming we stick with the system we've been using, the way that we'd share secret information with a hosting service like Heroku is by establishing a secure, private line of communication between us and the host, and transferring the secrets through that protected channel. We'll look at this more next week when we start deploying to Heroku.

#### Your Turn :: Paperclip with S3

In your groups, go back to your IMDB clone and switch it over from local storage to using S3.

## Additional References
- [Paperclip's GitHub Page](https://github.com/thoughtbot/paperclip)
- [Heroku's Paperclip + S3 Walkthrough](https://devcenter.heroku.com/articles/paperclip-s3)
- [Adding Environmental Variables to Heroku (For When You Deploy)](https://devcenter.heroku.com/articles/config-vars)
- [RubyDocs Page on Paperclip](http://www.rubydoc.info/gems/paperclip/Paperclip)
- [RubyDocs Page on using S3 with Paperclip](http://www.rubydoc.info/gems/paperclip/Paperclip/Storage/S3)
