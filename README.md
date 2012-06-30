# Jenkins Rails Template

Used at work.

## Install Jenkins and prepare Ruby Environment

```shell
# Jenkins:

wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | apt-key add -
# Then add the following entry in your /etc/apt/sources.list:
echo "deb http://pkg.jenkins-ci.org/debian binary/" >> /etc/apt/sources.list
apt-get update
apt-get install jenkins

# Jenkins will run on port :8080


# Ubuntu RVM Reqs
/usr/bin/apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion

# sphinx & mysql
apt-get install libmysql++-dev libmysqlclient-dev mysql-server libmysql-ruby libmysqlclient-dev

# capybara-webkit
apt-get install libqt4-dev libqtwebkit-dev
# redis
apt-get install redis-server

# sphinx compiling
cd /usr/local/src
wget http://sphinxsearch.com/files/sphinx-2.0.4-release.tar.gz
tar xf sphinx-2.0.4-release.tar.gz
cd sphinx-2.0.4-release
./configure && make && make install


sudo su - jenkins
curl -L https://get.rvm.io | bash -s stable --ruby
rvm use 1.9.3 --default
ssh-keygen

cat /var/lib/jenkins/.ssh/id_rsa.pub
# add public key to git-Server
# and capistrano deploy user (if necessary)
# -> cat >> .ssh/authorized\_keys

source ~/.rvm/bin/rvm

# set git environment for jenkins
git config --global user.email "developers@pludoni.de"
git config --global user.name "Jenkins"
```


* Sphinx will run on port 8080. Change it, by changing port in /etc/default/jenkins
* First, enable security for jenkins
* press "Check now" in bottom right of plugin->Advanced to get a list of all plugins

Useful plugins:
* Jenkins GIT plugin
* HTML Publisher plugin -> easily save simple-cov, yarddoc, rails best practices output
* Jenkins instant-messaging plugin -> required for Jabber
* Jenkins Jabber notifier plugin
* Green Balls -> successful builds are green, not blue
* Simple Theme Plugin -> provides custom css to make the enhance design, font-sizes etc.
* AnsiColor -> Console will output colored (rspec etc)
* Brakeman Plugin -> collects brakeman reports over time
* Jenkins ruby metrics plugin -> a little outdated, mostly only useful for Ruby 1.8. But with some modifications, it can collect coverage statistics (and fail build upon) and basic code stats (LOCs, methods per class etc)
* Rvm
* Jenkins Wall Display Master Project -> special view for wall monitor

probably useful, but have not used:
* Jenkins Rake plugin
* ruby-runtime

## Configuration

/configure:
* Jabber Notification:
  * Jabber ID: name@server.de
  * Initial group chats: room@conference.server.de
* E-Mail: SMTP, Sender-Email, Username, Password

Styling:
* put css under /var/lib/jenkins/userContent and it will also be available under http://server/userContent

Rails Job:

* clone this repository to /var/lib/jenkins/jobs/RailsTemplate
* Create a new job by copying this via the interface
* Adjust git-path, jabber-room or anything else in the file

## Necessary Gems

Put necessary gems in your project's Gemfile:
```ruby
group :test, :development do
  gem "yard"
  gem "redcarpet"   # for yard to process .md files
  gem "rails_best_practices"
  gem 'simplecov', :require => false
  gem 'simplecov-rcov', :require => false
  gem "ci_reporter", require: false
end
```

Modify your spec/spec_helper:
```ruby

# almost at top or inside spork.prefork block, if you are using sport
  unless ENV['DRB']
    require 'simplecov'
    require 'simplecov-rcov'
    class SimpleCov::Formatter::MergedFormatter
      def format(result)
        SimpleCov::Formatter::HTMLFormatter.new.format(result)
        SimpleCov::Formatter::RcovFormatter.new.format(result)
      end
    end
    SimpleCov.formatter = SimpleCov::Formatter::MergedFormatter
  end
  #. .... later in rspec configure

RSpec.configure do |config|
  config.tty = true  # enable color for rspec
#...
```

Make sure, the specs can be run as "rake specs", this is required for working properly with the ci_reporter gem. Modify your Rakefile like that:
```ruby
# Rakefile
require File.expand_path('../config/application', __FILE__)

unless Rails.env.production?
  require 'ci/reporter/rake/rspec'
  require 'rspec/core/rake_task'
end
```

Then the build should be possible.

## Build Process Explained

1. Git-Checkout, Bundle, Migrate, Test prepare
2. We are using simple-cov to generate rcov-style coverage information.
3. Runs specs
3. ci-reporter helps generating junit-style xml-files, so Jenkins can process these directly and we get a detailed overview over failing tests.
3. yard generates documentation
3. brakeman will generate security report
3. Rails Best Practices will run
3. cap deploy will be run, to put the passing build live
3. Publisher collect and archives reports
3. At the end, the bot reports success to the chat





