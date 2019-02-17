

# EnhancedSnake for JavaScript & React Native Boot Camp

For this homework assignment, you are tasked with finishing the implementation
of this app. To get started, click `fork` on the top right corner of this page
to get your own copy of the repository.

## Setup

### Downloading the new repository

Using the command line, navigate to the folder containing your React Native
projects, then run `git clone` followed by your forked repository's URL. As an
example, I can run `git clone https://github.com/Dallas-J/EnhancedSnake` to
download my repository (don't use mine though, use yours)!

Once you have the repository downloaded, `cd` into the new folder and run
`npm install` to install dependencies.

### Modifying your development environment

Though you all should have functional development environments, we will want to
do a few things differently to ensure that our applications perform well.

First, run `npm install --global exp` from the command line. This will install
Expo's dedicated command line development tool. Now, from the project directory,
use `exp start --no-dev` in place of `npm start`. This command starts your
project in production mode, which is very important if you are running a
quick-to-update app (most games fall into this category).

The last thing you will need to do is go to [Expo's website](https://expo.io)
and create an account. You will be prompted to enter your login details for your
Expo account the first time you run `exp start --no-dev`.

## What's Left Out?

The following functionality is currently left unimplemented:
* Scoring
* Moving
* Losing
* Pausing

These are listed in order of difficulty to implement, so I would suggest you
start at the top and see how many things you can implement. You need to add code
to complete any of these solutions, but you should not have to modify my code
until you begin to implement pause functionality.

## Wrapping Up

The code itself contains comments to help you understand everything, whether you
need to edit it or not. If you're having a problem understanding any of the
code, try modifying it to see what it changes. I also encourage you all to ask
and answer as many questions as you can on the class Slack channel. I will be
checking it frequently to see how everyone is doing!

<br>
<br>
##Submit Solution
---
<div class="jumbotron jumbotron-fluid bg-light text-dark">
  <div class="container">
    <form>
        <div class="row">
            <div class="col">
            <input type="text" class="form-control" placeholder="Name">
            </div>
        </div>
        <br>
        <div class="form-group">
            <input type="email" class="form-control" id="exampleFormControlInput1" placeholder="name@example.com">
        </div>
        <div class="form-group">
            <label for="exampleFormControlTextarea1">Comments (Optional):</label>
            <textarea class="form-control" id="exampleFormControlTextarea1" rows="3"></textarea>
        </div>
        <div class="form-group">
            <input type="file" class="form-control-file" id="exampleFormControlFile1">
        </div>
        <button type="submit" class="btn btn-primary">Submit</button>
    </form>
  </div>
</div>