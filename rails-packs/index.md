<!-- .slide: data-background="#1A53aE" -->

# Rails "Packs"

Stefan Wienert - Ruby Frankfurt

2024-06-16

----

## Growing a Rails app

![](https://documents.pludoni.de/uploads/b6b78e9f5675c19e2c9450802.png)

Greenfield app: add models, controller

----

## App grows vertically 

Each feature gets more complex...

<div class='fragment'>
Solution: Maybe add some more concepts?

- Services, other ``app/*`` stuff
- hexagonal architecture / DDD
</div>


----

## App grows also horizontally 

App has more and more distinct parts!

- Auth, User Management
- Billing
- Admin 
- Core Business
- Import/Export 
- Analytics Statistics...

----

Everthing in one folder!

```
find app/models  -type f | wc -l

262
```

----

## App folder is hard to grok!

Solution: **Microservices**!

<p class='fragment'>
Just kidding! 🧌
</p>


----

## App folder is hard to grok!

Solution:  **Namespaces!**

----

## feature is spread out now...


```
find billing app/

- app/models/billing/bill.rb
- app/models/billing/booking.rb
- app/models/billing/payment.rb
- app/controllers/billing/bills_controller.rb
- app/controllers/billing/payments_controller.rb
- app/views/billing/bills/index.html.erb
- ...
- app/jobs/billing/...
- app/javascript/billing/
- test/controllers/billing/...
```

+ dozens of lines in ``config/routes``

----

## One feature is spread out and hard to load into your head!

Solution: Engines!

----

## Engines

Engines are "reusable"* by design!

<ul>
    
<li class='fragment'>Need to be a Gem <small>(That's only used once)</small> 
<li class='fragment'>Lot of new concepts for isolation: 
    <small>Isolated namespace? Mountable? Dummy App, "main_app"?</small>
<li class='fragment'>Hard to communicate with outside
    <small> using style/layout of main app, accessing "current_user"/"User" from Main App etc... </small>
<li class='fragment'>World of pain! 🔥
</ul>
    
<small>
*  Before software can be reusable it first has to be usable
</small>

----

Engines are nice, but we don't need the complexity of allowing reusability!

----

Solution: **Packs**, as used by Shopify, Gusto

Pack: light weight mini Rails app skeleton, that may depend on other packs (or the "main app", but is just part of the Rails app)


----

Different flavors and tools:

- packs
<small>Base Pack 'spec' (internal Gem)</small>
- packs-rails
<small>opionionated folder structure, everything in packs/billing/, set's up Rails to load everything transparently</small>
- packwerk
<small>define visibility rules between constants and packs; Validate them (Shopify)</small>

----

<!-- .slide: data-background="#1A53aE" data-slide="image" -->


<small>
https://github.com/rubyatscale 😱
</small>

![](https://documents.pludoni.de/uploads/b6b78e9f5675c19e2c9450803.png)


----

Getting started:

```bash
bundle add packs-rails packs packwerk
bin/packwerk init

mkdir -p packs/billing/app/models/billing/
mkdir -p packs/billing/app/controller/billing/
mkdir -p packs/billing/app/views/billing/
...
# or use their micro generator
bundle exec packs create packs/billing
```

Done! Autoloading "just works"™️

----

packs-rails offers solutions for:

- Composable routes file (Since Rails 7) 
```ruby 
# config/routes.rb
Rails.application.routes.draw do
  draw(:billing) 
  # inlines ./packs/billing/config/routes/billing.rb
```
- RSpec/Test-Unit loader "just run all the billing specs": 
```ruby 
# .rspec 
--colour
--require rails_helper
--require packs/rails/rspec
```

----

#### Our Recipies

- Vite/Typescript aliases
```javascript
import BillingApp from 'billing/components/App.vue'
```
- I18n + [I18n-Tasks](https://github.com/glebm/i18n-tasks)  
<small>``t('billing.*')`` -> ``packs/billing/config/locales/de.yml`` </small>

- ``rails generate pack billing`` 
<small>-> Better generator, that generates a whole pack-stub, adds routes etc.</small>

----

<!-- .slide: data-background="#1A53aE" data-size="summary" -->

<ul>
<li class="pro">Our App: New features are now often new packs (about 10 packs at the moment)</li>

<li class="pro">Everytime you have the "New App Smell"! ✨🤩✨
<li class="pro fragment" data-fragment-index="1">Easier to load a feature into your head, not overwhelmed with hundreds of files.
<li class="pro fragment" data-fragment-index="1">Run all related tests easily
<li class="pro fragment">easy to start - just make a new pack for the new feature
<li class="neg fragment">Tooling/Editors maybe not familiar with the folders (Rails.vim)
<li class="fragment">❓ TBD: Packwerk validations, refactoring of "Main App"</li>

</ul>

----

#### Links:

<small>

- [Scaling our Ruby on RailsMonolith using Packwerk](https://medium.com/pennylane-engineering/scaling-our-ruby-on-rails-monolith-using-packwerk-part-1-b787aaa218ff)
- [rubyatscale/packs-rails](https://github.com/rubyatscale/packs-rails)
- [engineering.gustom: How To Guide to Ruby Packs Gustos Gem Ecosys](https://engineering.gusto.com/a-how-to-guide-to-ruby-packs-gustos-gem-ecosystem-for-modularizing-ruby-applications/)

</small>

--

    
<small style='display:flex; gap: 15px; font-size: 0.4em; justify-content: center'>
<a rel="me" href="https://ruby.social/@zealot128" title="Follow me on Mastodon">
masto: @zealot128@ruby.social
</a>
<a rel="me" href="https://www.stefanwienert.de">
blog: stefanwienert.de
</a>
<a rel="me" href="https://www.linkedin.com/in/stefanwienert/">
Linkedin: stefanwienert
</a>
<a rel="me" href="https://github.com/zealot128">
gh: zealot128
</a>
</small>

