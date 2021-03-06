dev+ops guidelines
==================

Coding &amp; general guidelines for in-house projects

# Intro

This document presents general or specific rules that *must* or *should* be observed
when coding/deploying et ERASME.


If some points are not listed here, send a PR.
If you disagree with some points, open an [issue](https://github.com/erasme/guidelines/issues).

# Applications & Code

The preferred application stack is ruby based. Thus, this section will
mostly focus on Ruby applications stacks.

## Licence

TBW.

## Source code control

All code must be under source code control using git.

Public projects must be on github.

Projects must have at least two branches :
- `master` (the stable, deployable branch)
- `devel` (unstable branch).

## Ruby environment

### Testing

Tests must reside in a `spec/` directory, located in the application
root.

While TDD is not strictly required, tests must be written. It's the
developper responsability to explain upstream that she can't "cram in
this shiny new feature" because she "has to improve test coverage".

Test coverage must be in place.

The coverage method should use SimpleCov.

A travis config file and associated hooks should be setup for each
project repository.

### Implementation

- Produced code must target MRI Ruby >= 2.1
- Code should run on jruby 1.7+

### Environment setup

While each developpement can use it's ruby environment of choice, the
deployment environment is [`rbenv`](https://github.com/sstephenson/rbenv).

### Gemfiles

Gemfiles must use a HTTPS uri :

```
source 'https://rubygems.org'
```

`source :rubygems` is deprecated.

An application must come with a `Gemfile.lock`.

However, if specific / minimal versions of some Gem are required, they
must be set un Gemfile.

Example minimal `Gemfile`:

```
source 'https://rubygems.org'

# Distribute app as a gem
# gemspec

# Optional JSON codec (faster performance)
# gem 'oj'

# Project requirements
gem 'rake'
gem 'dotenv'

# Component requirements
gem 'bcrypt'
gem 'mysql2'
gem 'sequel'
gem 'redis-rack'

# Server
gem 'puma'

# Redis
gem "redis", "~> 3.0.1"
gem "hiredis", "~> 0.4.5"

group :development do
  gem 'awesome_print'
  gem 'stackprof'
  gem 'pry'
  gem 'guard'
  gem 'guard-rspec'
end

# Test requirements
group :test do
  gem 'rspec'
  gem 'factory_girl'
  gem 'rack-test', :require => 'rack/test'
  gem 'simplecov'
end
```


### Web server

The web server currently in use is
[`puma`](https://github.com/puma/puma).

Thus, a `gem 'puma'` entry must be present in the application Gemfile.

A config file for thin must be provided in the `config/` directory located
in the application root, and must be named `puma.rb`.

If the application requires another webserver, this issue should be
discussed in the team before

### Rake tasks

A `Rakefile` must be provided.

Rake tasks must be defined in the `tasks/` directory located in the
application root.

Tasks must be grouped by topic (e.g. `server`, `db`).

Each topic must be defined in `tasks/topic.rake`, using the following
scheme :

```rakefile
namespace :db do
  task :migrate do
  ...
  end
end
```

#### db namespace

Regarding the `db` namespace, migrations tasks must be triggered by a
target name `db:migrate`.

The `db:migrate` task must create a database dump if an upgrade is required.

A `db:migrate_check` task must be present that exits with non-zero code
if some migrations are pending.

Here is an example of such task :

```rakefile
desc 'Check migrations'
task check: :migrate_check do
  Sequel.extension :migration
  Sequel::Migrator.check_current(Sequel::Model.db, 'db/migrate')
end
```

## Code

Ruby code must use a 2 spaces indentation. Tabs must not be used and must be  to 2 spaces :

### In Vim

```vimrc
:set tabstop=2
:set shiftwidth=4
:set expandtab
```

### In sublime-text

```json
{
  "tab_size": 2,
  "translate_tabs_to_spaces": true,
}
```

### Rubocop configuration

Copy config_files/rubocop.yml to ~/.rubocop.yml

### JSHint configuration

Copy config_files/jshintrc to ~/.jshintrc

### Exceptions

Exceptions handling must be qualified, and not catch-all (`rescue` which catches `StandardError`, or `rescue Exception` which catches everything) style.
A log with a clear and understandable message must be emitted using the logger when an exception occurs.

### Leverage caching

Use Redis for shared caching.

### Sessions

Sessions must be shareable. The best way to is to use `redis-rack`.

# System & Deploys

## Deploys

Deploys must :
- be fully automated (but manually triggered)
- be idempotent
- be repeatable
- use dependencies

Deploys should be:
- hardware independent

Deployment scenarios must use Ansible v1.6+ roles.

Deployement scenarios must be based solely on information found in
project's README.md

If a scenario can not be written following the information provided in
the README.md file, this file must be updated or fixed.

## Running environments

Available running environments must include at least :

- development
- production
- test

Other alternative names (e.g. `dev`, `prod`) must not be used for consistency reasons.

However, additional names can be used (and documented) if needed.

## Config files

Config files must live in the config/ directory located at application
root.

No other file must reside in this directory.

Ruby config files must end with `.rb`.

YAML config files extention must be `.yml`

Real config files should not be commited in the source repository, and a
properly commented `some_config.rb.sample` must be provided instead.

When a `.sample` file changes (added/changed/removed configuration
keys), the developper must inform the person automating deploys OR
change the deployment scripts to account for this change.

Thus, deployment tasks must generate proper configuration files.

Inputs coming from configuration files must be sanitized (whitespace
trimming, slash-deduplication for URLs, etc...).

## Logging

All logging must be done under `/var/log/<application_name>`. Thus, a
proper entry for the `thin.yml` could be :

```yaml
log: /var/log/my_awesome_app/thin.log
```

Code should not print to directly to filedescriptors (`puts`, `p`, `pp`, ...) but use a logger instead.

## Production code

Code deploys are automated and must be done from git checkouts.

The stable branch must be the master branch and master must always be deployable.

Code and configurations must not be edited by hand on production servers.

Code and configurations should not be edited by hand on staging servers, except for debugging/testing purpose.
