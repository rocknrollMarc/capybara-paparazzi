# Capybara::Paparazzi

Capybara::Paparazzi simplifies and automates the task of capturing
screenshots of your app by snapping full-length photos of your pages
in various window widths during a run of your test suite. If you've
ever wanted to get screenshots of your user flows to check how
everything looks on a phone vs. on a tablet vs. on a desktop, this gem
is for you. This is particularly useful if you are using responsive
CSS. Additionally, it draws a line on the screenshot where the "fold"
would be, showing a user's first impression of a page.

Capybara::Paparazzi hooks into Capybara's feature tests and takes
pictures at significant user actions (like visiting a page, clicking
buttons, etc.). It relies on a driver that supports saving screenshots
and executing javascript, and was originally built with Poltergeist in
mind (due to its headless nature).

A few driving principles of Capybara::Paparazzi:

- **Responsive**: It can take screenshots in whatever sizes you want.
- **Unobtrusive**: You shouldn't need to edit your feature test scenarios
  to get screenshots.
- **Easy**: Simple to set up, and out-of-the-box settings should get you far.
- **Customizable**: You should be able to tweak it to your needs easily,
  without needing to re-invent too much to achieve what you want.
- **Exhaustive**: Taking extra screenshots is better than missing out on
  important things. It's easy to delete or ignore unnecessary files later.

## Installation

Add a line to your application's Gemfile:

    group :test do
      gem 'rspec-rails'
      gem 'capybara'
      gem 'poltergeist'
      gem 'capybara-paparazzi' # <-- add this one
    end

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install capybara-paparazzi

## Usage

### Quick-Start

After getting the gem installed, using Capybara::Paparazzi is pretty simple.
In your test setup file (using rspec, probably `spec/spec_helper.rb`),
add something like this:

```ruby
# Other rspec & capybara setup, such as:
# require 'rspec/rails'
# require 'capybara/rspec'

require 'capybara/poltergeist' # <-- Use a fancy driver.
Capybara.default_driver = :poltergeist # <-- Set it to be used for everything.

Capybara::Paparazzi.follow(:poltergeist) # <-- Take screenshots of everything the driver does.

```

Now just run

    $ rspec

or

    $ rake

and you'll get screenshots in a `screenshots` folder.

### Configuring

Use `Capybara::Paparazzi.follow(<driver_name>, ...)` to take screenshots
for a particular driver. You can also do `config.follow(<driver_name>, ...)`
inside of a configuration block, instead. It doesn't matter if this
happens before or after the driver itself is actually registered with
Capybara, just as long as the name matches.

Here's a fairly complete list of things you can do:

```ruby
Capybara::Paparazzi.config do |config|
  config.js_var_name = 'MY_FOLD_DIV' # Name of the div used to draw the fold line.
  config.js_var_style.merge!(color: 'red', fontWeight: 'bold')
  config.screenshot_sizes = [ config.screenshot_sizes.first, (config.screenshot_sizes.last + [ :EXTRA_DATA ]) ]
  config.file_dir = '../screenshots'
  # config.make_fewer_directories = true
  # config.make_shorter_filenames = true
  config.path_and_suffix_generator = ->(shooter, url) {
    shooter.path_and_suffix_for_url(url).collect{|p| p.split('/').last }
  }
  # config.js_setup_script = ->(shooter){ ... }
  config.js_resize_script = ->(shooter, height) {
    "#{shooter.js_var_name}.style.height = '#{height}px'; MY_FOLD_DIV.textContent = #{shooter.screenshot_size.inspect.to_json}; MY_FOLD_DIV.style.fontSize = '#{height/4}px';"
  }
  # config.js_cleanup_script = ->(shooter){}
  config.before_save_callback = ->(shooter) {
    shooter.resize_window(shooter.width * 2, shooter.height / 2)
    shooter.do_shot = shooter.suffix.to_s.empty?
  }
  config.follow(:poltergeist)
end
```

For the default settings, see [shooter.rb](https://github.com/sbull/capybara-paparazzi/blob/master/lib/capybara/paparazzi/shooter.rb).

### Skipping Screenshots

Many test flows might have common steps that are just plain redundant
to keep capturing. Disabling Capybara::Paparazzi for those flows could
be useful. Here's how:
```ruby
without_snapshots do
  visit root_path # <-- your stuff
end
```

`without_snapshots` is also known as `without_paparazzi`.

You can also disable Capybara::Paparazzi in scenarios by calling

    turn_snapshots_off

(aka `turn_paparazzi_off`), but then you'll have to turn it back on
for future scenarios by calling

    turn_snapshots_on

(aka `turn_paparazzi_on`).

### Manual Screenshots

Try as it might, Capybara::Paparazzi could miss some photos that you
want it to take, particularly if you're using javascript to change
the page without navigating to a different page. You can call

    take_snapshots

in your scenarios to manually capture the page.
(`take_snapshots` is also known as `paparazzi_take_snapshots`,
in case you use something else that defines `take_snapshots` differently.)

You can leave these calls in your code even when you're not using
`#follow`, and they will be no-ops.

### Screenshot Sizes

Capybara::Paparazzi uses default screen sizes to represent common
phone, tablet, and desktop browser dimensions. The most important
feature is the width. The height is used to draw the "fold" line - the
default sizes attempt to take into account the browser chrome that
gets added to the top and bottom of browser windows, which results in
a shorter view than the actual device height. As indicated above,
you can change the screenshot sizes with

    config.screenshot_sizes = [ [ 1920, 1080, 'my big', 'screen' ] ]

The first 2 values in the array are width and height, but the whole array
is made available for use in customized configuration scripts.

### Filenames

By default, a new directory is created to contain the screenshots for
each distinct URL. There are a few ways to change this if it's not to
your liking:

1. Use `config.make_fewer_directories = true`, which will skip the last
   directory level and put screenshots for multiple pages in the same
   directory.

2. Use `config.make_shorter_filenames = true`, which causes the basenames
   of the files not to include a representation of the full path
   (the full path in the basename makes it easier to understand when
   copied/moved outside of the screenshots directory tree).

3. Override `config.path_and_suffix_generator = ->(shooter, url) {...}`,
   to return `[path, suffix]` (`suffix` indicates repeated screenshots
   of the same url).

4. Use `config.before_save_callback = ->(shooter) {...}` to change
   `shooter.filename`, in which you can access `shooter.screenshot_size`
   and other attributes.


## Tips & Tricks

### Turning Capybara::Paparazzi On and Off

Use an environment variable to turn screenshots on or off.
You probably don't want them running all the time, and only
need them on occasion:
```ruby
if ENV['CAPYBARA_PAPARAZZI_DIR'].present?
  Capybara::Paparazzi.config do |config|
    config.file_dir = ENV['CAPYBARA_PAPARAZZI_DIR']
    config.follow(Capybara.default_driver, Capybara.javascript_driver)
  end
end
```

### Driver Setup

Poltergeist is a great headless driver to use, particularly due to its
ease of installation.
Here's a simple approach that could work for you:

1. Install [PhantomJS](http://phantomjs.org/).
  1. Prereqs: install [nodejs](http://nodejs.org/) and npm.
     Try something like: `sudo apt-get install nodejs`
  2. Use npm to install phantomjs: `sudo npm install -g phantomjs`
2. Install [Poltergeist](https://github.com/teampoltergeist/poltergeist).
  1. Add poltergeist to your Gemfile: `gem 'poltergeist'`
  2. `bundle install`
3. Use Poltergeist:
  1. `require 'capybara/poltergeist'`
  2. `Capybara.javascript_driver = :poltergeist`

## TODO

- The photo-capturing triggers may be incomplete, and enhancements may be
needed to be sure to capture everything. Currently, `visit` and `click`
are the primary triggers.

### Compatibility

Original development was done in the following environment:
- capybara 2.2.1
- poltergeist 1.5.1
- phantomjs 1.9.7
- ruby 2.0.0
- ubuntu 12.04

Capybara::Paparazzi should work with other drivers and platforms.
It is known to work on OSX, using both poltergeist and
[webkit](https://github.com/thoughtbot/capybara-webkit) as drivers.

Please open a github issue if you discover a problem with another
platform or driver.

## Related Projects

- [Capybara](https://github.com/jnicklas/capybara)
- [Capybara::Animate](https://github.com/cpb/capybara-animate) - makes animated .gifs of scenarios.
- [capybara-screenshot](https://github.com/mattheworiordan/capybara-screenshot) - takes screenshots of failing scenarios.
- [Headless](https://github.com/leonid-shevtsov/headless) - has ways to capture screenshots and video.

## Contributing

### Generic Github Instructions

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

### Working on a Gem

You can read up on how to develop a local gem on
[rubygems.org](http://guides.rubygems.org/make-your-own-gem/).

Here's the TL;DR:

1. Make your edits.
2. Update the version (lib/capybara/paparazzi/version.rb), say to `0.0.1.new`.
3. `gem build capybara-paparazzi.gemspec`
4. `gem install ./capybara-paparazzi-0.0.1.new.gem`
5. In your project that uses Capybara::Paparazzi:
  1. Update your Gemfile to say `gem 'capybara-paparazzi', '0.0.1.new'
  2. `bundle install`

Some useful one-liners for the above:

In `capybara-paparazzi` repository:

    gem uninstall capybara-paparazzi && gem build capybara-paparazzi.gemspec && gem install ./capybara-paparazzi-0.0.1.new.gem

In your project:

    rm vendor/cache/capybara-paparazzi-0.0.1.new.gem && bundle
