### HOW DO I GET STARTED WITH MY SINATRA PORTFOLIO PROJECT?

#### DO NOT WRITE CODE YET!!!! ####

###### Pre-coding work
1.  Ideate!  What do you want to build?
  - choose a domain you're familiar with!
  - choose a domain you care about
2.  Wireframing & User Stories
  - Write down your models, their attributes, and their associations
  - As a user, I can .....
  - A user should be able to .....
  - What does your app _do_?
3.  Design your MVP = 'Minimum Viable Product' vs. what are my 'stretch goals'
  - Stretch goals - bonus features you want but don't need

#### NOW, WE CODE! * but NO controllers or views yet *

4.  Build your models
  - Migrations
  - Model classes
  - Associations (& validations)

5. Test your models and the associations in the console
  - create some seed data
  - adjust migrations as needed

#### NOW, CONSIDER CONTROLLERS AND VIEWS

6. Start with your ApplicationController helpers - #logged_in? and #current_user
  - add your login/signup/signout routes

7. Build out controller routes for other models (add a controller for each model)

8. Build views and controller actions based on the flow of your app, one step at a time, testing as you go!
  - Use shotgun and pry (or raise/inspect) all the time!


###### Using the corneal gem

You are welcome to use the corneal gem.  However, you should understand what it's doing.  Remove any folders and files you're not using.  For example, if you're not going to write any tests, delete the `spec` folder.
