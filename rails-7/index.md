
# Rails 7 & 7.1pre Features

Stefan Wienert

2023-01-25

----

## My Rails history

- Rails Dev since Rails 2.3/Ruby 1.8.7
- Same Company for 12 years, running ~20 Rails-apps in prod.
    - Oldest started from 3.0
    - 6 on 7.0, 12 on 6.1

----

<!-- .slide: data-background="#bd9adb" -->

## Schedule:

1. Rails 7
2. New Asset Pipeline®
3. Rails 7.1pre
4. Upgrade Strategies/Discussion

Note:
    Disclaimer: there will be a lot of slides with new features. Some things are "nice" but not immediately useful. I also made a selection of features I use/happy to have. There are many more PR/commits that I don't include, as well as tons of fixes


----

## Questions to the Audience

<ul>
<li class='fragment'>
Who has at least one production level app on Rails? </li>
<li class='fragment'>
Who has at least one production level app on < Rails 6 (5.x 4.x 3)</li>

<li class='fragment'>
Who knows, what "Dual Booting Rails" means?</li>
<li class='fragment'>
Who uses it?</li>
</ul>

---

<!-- .slide: data-background="#1A53aE" -->

## Rails 7

- Released on: December 15, 2021
- 6.1 will receive (low/medium) Sec-Fixes until 7.1 is released
- [Diff to 6.1](https://github.com/rails/rails/compare/v6.1.4.1...7-0-stable): 

```
git diff v6.1.4.1...origin/7-0-stable --shortstat
 2109 files changed, 83193 insertions(+), 36975 deletions(-)

4683 commits by 29 contributors
```




----

### Attribute-level Encryption

```ruby
class Person < ApplicationRecord
  encrypts :name
  encrypts :email_address, deterministic: true
end
```

- TLDR: declare in model
- use `deterministic`, if you need to look up names in SQL (email)
- can migrate/use unencrypted columns, will transparently encrypt when saving


----

| Content to encrypt                                | Original column size | Recommended size | 
| ----------------------------- | -------------------- | --------------------------------- | 
| Email addresses                                   | string(255)          | string(510)                       |
| Short sequence of emojis                          | string(255)          | string(1020)                      | 
| Summary of texts written in non-western alphabets | string(500)          | string(2000)                      | 
| Arbitrary long text                               | text                 | text                              |


<small>=> ~2x limit sizes of varchar columns to accommodate for Base64 key overhead and key</small>
 
[:arrow_right:Guide](https://edgeguides.rubyonrails.org/active_record_encryption.html) [PR](https://github.com/rails/rails/issues/41833)

----


### Strict loading


```ruby [2|3|5-6]
class User < ApplicationRecord
  has_many :bookmarks
  has_many :articles, strict_loading: true
end

user = User.first
user.articles     # BOOM
```

Disallow N+1 Loading on a model, attribute or global basis

[PR](https://github.com/rails/rails/issues/41181)

----

### load_async


```ruby
def index
  @categories = Category.some_complex_scope.load_async
  @posts = Post.some_complex_scope.load_async
end

- @posts.first #-> will block
```

Parallelize loading of lots of records

----

Keep in mind:

- overhead is non neglectable 
- Thread pool size for database connections
- Memory usage, if loading tons of records still high

[PR](https://github.com/rails/rails/issues/41372)

----

### In order of

```ruby

Post.in_order_of(:id, [3, 5, 1])
# SELECT "posts".* FROM "posts" 
# ORDER BY CASE "posts"."id" 
#   WHEN 3 THEN 1 WHEN 5 THEN 2 WHEN 1 THEN 3 
#    ELSE 4 END ASC
Post.in_order_of(:type, %w[Draft Published Archived]).
    order(:created_at)
# ORDER BY
#  FIELD(posts.type, 'Archived', 'Published', 'Draft') DESC,
#  posts.created_at ASC
```

(e.g. you have the order from another service, Elasticsearch, Machine learning)

[PR](https://github.com/rails/rails/pull/42061)

----

### `scope.excluding(*records)`

excludes the specified record (or collection of records) from
the resulting relation:

```ruby
Post.excluding(post)
Post.excluding(post_one, post_two)

class JobAd
  def related_jobs
    organisation.job_ads.excluding(self)
  end
end
```

[PR](https://github.com/rails/rails/issues/41439)


----

###  where.associated 

check for the presence of an association

```ruby
# Before:
account.users.joins(:contact).where.not(contact_id: nil)

# After:
account.users.where.associated(:contact)
```

Mirrors existing ``where.missing``.

----

### Enumerating Columns

<small>Before:</small>

```ruby
Book.limit(5)
# SELECT * FROM books LIMIT 5
```

<small>After:</small>

```ruby
Rails.application.config.
    active_record.enumerate_columns_in_select_statements = true
Book.limit(5)
# SELECT title, author FROM books LIMIT 5
```

Why: Avoid `PreparedStatementCacheExpired`, consistent column ordering.

[PR](https://github.com/rails/rails/issues/41718)

----

### virtual columns

PG only:

```ruby
create_table :users do |t|
  t.string :name
  t.virtual :name_upcased, type: :string, 
    as: 'upper(name)', stored: true
end
```

[PR](https://github.com/rails/rails/issues/41856)

----

### Unpermitted redirect

Raise error on unpermitted open redirects.

```ruby
redirect_to @organisation.website_url #OLD # BOOM

redirect_to @organisation.website_url, 
  allow_other_host: true # NEW
```

[Commit](https://github.com/rails/rails/commit/5e93cff83599833380b4cb3d99c020b5efc7dd96)

----

### Zeitwerk 

Zeitwerk Autoloader Mode is default and not changeable.

:arrow_right: All classes/modules must match file-path, or define exception/collapses in Zeitwerk

[PR](https://github.com/rails/rails/issues/43097)

----

### Activestorage Variants

Add ability to use pre-defined variants.

```ruby
class User < ActiveRecord::Base
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize: "100x100"
    attachable.variant :medium, resize: "300x300", 
      monochrome: true
  end
end

<%= image_tag user.avatar.variant(:thumb) %>
```

[PR](https://github.com/rails/rails/issues/39135)

----

### ActiveStorage misc.

vips is new default, but `mini_magick` can still be used. [PR1](https://github.com/rails/rails/issues/42790):
```ruby 
# default:
config.active_storage.variant_processor = :vips.
# if want to keep imagemagick
config.active_storage.variant_processor = :mini_magick
```

blob expiration can be set individually per call <a href="https://github.com/rails/rails/issues/42410" style='right: 3ch'>PR2</a>:

```ruby
rails_blob_path(user.avatar, disposition: "attachment", 
    expires_in: 30.minutes)
```

----

- Previewing video files: FFMpeg args configurable 
- Scene detection for preview 
- Analyzer will determine if the file has audio and/or video data and prefills: ``metadata[:audio]`` and ``metadata[:video]`` 
- Audio analyzer added: duration + bitrate extracted 
- make `image_tag` globally lazy loading (free WebVitals®) 
  ```
  config.action_view.image_loading = "lazy"
  ```
<div class='ignore-pr'>
    
[PR](https://github.com/rails/rails/issues/42471) 
[PR](https://github.com/rails/rails/issues/42790) 
[PR](https://github.com/rails/rails/pull/42425) 
[PR](https://github.com/rails/rails/issues/40096)
    
</div>

---

<!-- .slide: data-background="#1A53aE" -->

### Asset Story

``Sprockets`` is mostly deprecated now. ``Webpacker`` is archived.

----


New Gems:

- **[importmap-rails](https://github.com/rails/importmap-rails)**: <small>Link to JS deps directly to CDN (or vendor), use browser as bundler using modules.</small>
- **[propshaft](https://github.com/rails/propshaft)**:
  <small>Successor of sprockets, but without any processing (static files, images)</small>
- **[jsbundling-rails](https://github.com/rails/jsbundling-rails)**: 
  <small>Installation scripts and very thin layer to wrap JS build tools (esbuild, rollup, also Webpacker)</small>
- **[cssbundling-rails](https://github.com/rails/cssbundling-rails/)**: 
  <small>Installation scripts for Tailwind, Bootstrap-scss</small>


----

### Upgrade:


```flow
st=>start: Rails 6 Upgrade
wb=>condition: Using Webpacker?
sprockets=>condition: Sprockets 
    (scss,coffee,
    es6,erb)?   
sh=>end: Keep using or switch 
     to shakapacker if newer
     Webpack requrired
propshaft=>end: Propshaft
pech=>end: Keep using sprockets for now
  Plan migration
st->wb
wb(yes)->sh
wb(no)->sprockets
sprockets(no@Only Static)->propshaft
sprockets(yes@Have assets)->pech
```


----



```flow
st=>start: New Rails 7 app

simple=>condition: Simple /
    only Turbo/Stimulus

importmaps=>end: Use Importmaps
managetools=>condition: Want to manage
 bundler myself
jsbundling=>end: Use JSbundling to
    integrate your tool (esbuild)
vitewebpack=>end: Use Shakapacker or Vite(!)
st->simple
simple(yes)->importmaps
simple(no)->managetools
managetools(yes)->jsbundling
managetools(no)->vitewebpack
    
```
    

---

## Rails 7.1 pre

> <small>The “minor” jumps between 4.0 -> 4.1 -> 4.2 etc. took an average of 273 days</small> 

-> IMO should be released soon :timer_clock: 

```
git diff origin/7-0-stable...origin/main --shortstat
 1735 files changed, 60562 insertions(+), 42135 deletions(-)

3584 commits by 56 contributors
```

<!-- .slide: data-background="#1A53aE" -->


----

### password challenge

```ruby
password_params = params.require(:user).permit(
  :password_challenge, :password, :password_confirmation,
).with_defaults(password_challenge: "")

# Important: MUST not be nil to activate the validation

if current_user.update(password_params)
  # ...
end
```


Requires `has_secure_password`, 
Safes some boilerplate and integrates nicely in the core validation flow.


[PR](https://github.com/rails/rails/issues/43688)

----

### generates_token_for


Automatically generate and validate various “tokens” for the user, think: Password Reset Token, Login Token etc.

```ruby [2-3|6]
class User < ActiveRecord::Base
  generates_token_for :password_reset, expires_in: 15.minutes do
    BCrypt::Password.new(password_digest).salt[-10..]
  end
end

user = User.first
token = user.generate_token_for(:password_reset)

User.find_by_token_for(:password_reset, token) # => user

user.update!(password: "new password")
User.find_by_token_for(:password_reset, token) # => nil
```

[PR](https://github.com/rails/rails/issues/44189)

----

### normalizes 

[PR](https://github.com/rails/rails/issues/43945)


```ruby
class User < ActiveRecord::Base
  normalizes :email, with: -> email { email.strip.downcase }
end

user = User.create(email: " CRUISE-CONTROL@EXAMPLE.COM\n")
user.email                  # => "cruise-control@example.com"

user = User.find_by(email: "\tCRUISE-CONTROL@EXAMPLE.COM ")
```

----

### select with hash values

FINALLY, ``ARRelation#select`` can be used with hash syntax, too.

```ruby
Post.joins(:comments).
  select(posts: [:id, :title, :created_at], comments: [:id, :body, :author_id])
Post.joins(:comments).
  # also with selection-aliases
  select(posts: { id: :post_id, title: :post_title }, comments: { id: :comment_id, body: :comment_body })
```

----

### User.authenticate_by

Instead of manually loading the user by mail and THEN validating the password:

```ruby
User.find_by(email: "...")&.authenticate("...")
```

Use the new method, which is also supposed to be timing-Attack resistant:

```ruby
User.authenticate_by(email: "...", password: "...")
```

[PR](https://github.com/rails/rails/issues/43880)

----

### Composite primary keys

Preliminary work has been merged, that allows Rails to better handle composite primary keys (on a later stage as noted in the PR)

```ruby
class Developer < ActiveRecord::Base
  query_constraints :company_id, :id
end
developer = Developer.first.update(name: "Bob")
# => UPDATE "developers" SET "name" = 'Bob' 
#    WHERE "developers"."company_id" = 1 
#      AND "developers"."id" = 1
```

[PR](https://github.com/rails/rails/pull/46331)

----

### Allow (ERB) templates to set strict locals.

Define, which locales (not controller instance vars) are required by a partial

```html
<%# locals: (message:) -%>
<!-- default -->
<%# locals: (message: "Hello, world!") -%>

<%= message %>
```

[PR](https://github.com/rails/rails/issues/45727)

----

### Find unused routes

```bash
rails routes --unused
```

Tries to find defined routes that either:
- without a controller or
- missing action AND missing template (implicit render)

----

### Postgres CTE Support


```ruby
Post.with(posts_with_comments: 
    Post.where("comments_count > ?", 0))

# WITH posts_with_comments AS (
#    SELECT * FROM posts WHERE (comments_count > 0)) 
#    SELECT * FROM posts
```

Note:
    Sometimes, it’s nice to define “Common Table Expressions” which are supported by some databases, to clean up huge SQL. In the past one had to fallback to raw SQL for this, but now it is easier to do it with AREL:

[PR](https://github.com/rails/rails/issues/45701)

----

### Limit log size

```
config.log_file_size = 100.megabytes
```

No gigantic  development.log or test.log anymore! 
🎉🍾
<small>(5GB in my case)</small>

[PR](https://github.com/rails/rails/issues/44888)

----

#### "ActiveDeployment"

### [rails/docked](https://github.com/rails/docked)



> Setting up Rails for the first time with all the dependencies necessary can be daunting for beginners. Docked Rails CLI uses a Docker image to make it much easier, requiring only Docker to be installed.

```bash
docked rails new weblog
cd weblog
docked rails generate scaffold post title:string body:text
docked rails db:migrate
docked rails server
```

----

#### "ActiveDeployment"

### Improved Docker asset building 


SECRET_KEY_BASE_DUMMY can be set to make assets:precompile not compile (Needed for docker)

```dockerfile
RUN SECRET_KEY_BASE_DUMMY=1 bundle exec rails assets:precompile
```

[PR](https://github.com/rails/rails/pull/46760)

----

#### "ActiveDeployment"

### Dockerfile (Prod)

Dockerfile .dockerignore entrypoint in newly generated apps

[PR](https://github.com/rails/rails/pull/46762)

> They're intended as a starting point for a production deploy of the application. Not intended for development


----

#### "ActiveDeployment" 
### [rails/mrsk](https://github.com/rails/mrsk)


> MRSK ships zero-downtime deploys of Rails apps packed as containers to any host. It uses the dynamic reverse-proxy Traefik to hold requests while the new application container is started and the old one is wound down. It works across multiple hosts at the same time, using SSHKit to execute commands.

---

### Upgrade strategies

<!-- .slide: data-background="#1A53aE" -->

<div style='text-align: left'>

I.
:    Use Rails Main for production <br><small>(Github, Basecamp, Shopify)</small>
    
II.
:    Dual Boot with next version  <br><small>([fastruby/next-rails](https://github.com/fastruby/next_rails), ([next-rails](https://github.com/fastruby/next_rails), [shopify/bootboot](https://github.com/shopify/bootboot))</small>

III. 
:    manual, planned upgrade :arrow_left: 

</div>

----

Manual Upgrade:

<div>
    
0. Use Bundle 2.4.0+ <small>to get better resolution results/errors ([PR](https://github.com/rubygems/rubygems/pull/5960) - released on christmas)</small>
1. Modify Rails version in Gemfile, <small>try ``bundle update rails ...`` with all gems  that require rails version </small>

</div>


3. Try boot the app, fix errors
4. use ``rails app:update``, or use [railsdiff.org](https://railsdiff.org/)
  
5. Fix tests + deprecations
  
<!-- .element: class="fragment" data-fragment-index="1" style="width: 100%" -->

----

6. Try enabling migration flags in 
    ```
    config/initializers/new_framework_defaults_7_0.rb
    ```
7. ([much] later) if all enabled, remove file, set 
    ```
    config.load_defaults 7.0
    ```

----


Rails Changelog

<small>Script/Page by me (with RSS-Feed) https://zealot128.github.io/rails-changelog</small>



<iframe border='0' style='border:0;height: 80vh;width: 100vw' src='https://zealot128.github.io/rails-changelog/versions/7-0'>
</iframe>

---

## Discussion

1. How frequent to you upgrade your Apps? 
2. How many releases do you stay behind?
3. Do you use Dual Boot or use Rails main-branch?
4. What's your biggest problems for updates?

--
masto: [@zealot128@ruby.social](https://ruby.social/web/@zealot128)
gh: [zealot128](https://github.com/zealot128), 
web: [stefanwienert.de](https://www.stefanwienert.de)

Company: pludoni GmbH 
<small>Job-boards: Empfehlungsbund.de JobsDaheim.de sanosax.de ITsax.de ITbbb.de ITrheinmain.de ...</small>
