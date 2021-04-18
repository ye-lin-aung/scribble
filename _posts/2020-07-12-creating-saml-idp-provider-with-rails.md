---
date: '2020-07-12 00:00:00'
title: Creating SAML IDP provider with Ruby on Rails
---

---
title: "Creating SAML IDP provider with Ruby on Rails."
date: 2020-07-02T7:48:30+06:30
author: Programmer Maung Wa
draft: false
tags:
- Ruby on Rails
- saml idp
- devise
---
Creating SAML IDP provider with Ruby on Rails.

# Rails SAML IDP Provider with devise

Recently I was working on single sign on solution for my company. After researching, we decided to use SAML for authentication. I believe everyone reading this post is already familiar with Rails and SAML. Security Assertion Markup Language, SAML is an open standard for exchanging authentication and authorization data between parties in XML format. It contains identity provider IDP which acts as a centralized authentication services and service providers SP use response XML from IDP to authenticate users. In simpler way, users logged in to IDP will automatically logged in to SP. We will have one Rails SAML identity provider with multiple SAML service providers connecting to it. In this post I will show you how to setup SAML IDP for Rails and shows how SAML SP can connect to it.

We are going to use devise for users in both SAML IDP and SAML SP since its the most common  authentication gem for Rails applications.

# SAML IDP with devise

### 1. Installing the Gem

To create Rails application that acts as SAML IDP. I am going to use gem called SAML IDP from [https://github.com/saml-idp/saml_idp](https://github.com/saml-idp/saml_idp). You need to add followings to your `Gemfile`. You  need to configure devise in your application. Make sure you have configured a working devise implementation in your Rails app.

```ruby
gem 'saml_idp'

# Devise
gem 'devise'
```

You can also use saml_idp gem for non Rails projects. You can find the documentation for non Rails project [here](https://github.com/saml-idp/saml_idp#not-using-rails).  After modifying the Gemfile, lets proceed to install the gems.

```ruby
bundle install
```

### 2. Generating a certificate

To setup an IDP, we need to generate x509 certificate. If you have openssl in you machine, using this command will generate a new certificate.

```bash
openssl req -x509 -sha256 -nodes -days 3650 -newkey rsa:2048 -keyout production.key -out production.crt
```

These keys are important, please keep them in a safe place.

### 3. Configuration IDP intializer

Ok, we have both key and cert generated in step 2. Lets add them to the configuration

```bash
SamlIdp.configure do |config|
  # We stored keys in environment variables.
  config.x509_certificate = ENV["SAML_CERT"]
  config.secret_key = ENV["SAML_SECRET_KEY"]
end
```

We still have a few configurations that we need to configure. We will come back to this later.

### 4. Modifying controller.

Create `saml_idp_controller.rb` in controllers folder and paste this code below

```ruby
class SamlIdpController < SamlIdp::IdpController
  # Devise authenticate user
  before_action :authenticate_user!, except: [:show]

  # override create and make sure to set both "GET" and "POST" requests to /saml/auth to #create
  def create
    if user_signed_in?
	# renderin xml response in saml format for devise's current_user
      @saml_response = idp_make_saml_response(current_user)
      render :template => "saml_idp/idp/saml_post", :layout => false
      return
    else
    # it shouldn't be possible to get here, but lets render 403 just in case
      render :status => :forbidden
    end
  end

  def idp_make_saml_response(found_user) # not using params intentionally
    encode_response found_user,{}
  end

  private :idp_make_saml_response

end
```

The code is simple, all it does it if service provider call `/saml/auth`  if user is signed in, it will response user object with SAML XML format, or else it will show devise's login page.

### 4. Routes

In `routes.rb`

```ruby

get '/saml/auth' => 'saml_idp#create'
post '/saml/auth' => 'saml_idp#create'
get '/saml/metadata' => 'saml_idp#show'
```

Make sure we are pointing the the same function in both get or post. By default get method to `/saml/auth` will go to `saml_idp#new` which will call `SamlIdp::IdpController` new method. This will render login page from the gem and it will show double login page, both devise login page and SAML login page. Pointing both get and post to `saml_idp#create` is required for devise based implementations .

### 5. Detail configurations

Now the only thing we left is configure the user fields as we need.

```bash
SamlIdp.configure do |config|
	# Domain of your identity provider. example, https://example.com
  base = ENV["HOST"]

  config.x509_certificate = ENV["SAML_CERT"]
  config.secret_key = ENV["SAML_SECRET_KEY"]
	# Name id formats
  config.name_id.formats = {
      uuid: -> (principal) { principal.id },
      persistent: -> (principal) { principal.email},
      email: -> (principal) { principal.email },
      name: -> (principal) { principal.name },
      department: -> (principal) { principal.department },
      position: -> (principal) { principal.position },
      user_type: -> (principal) { principal.user_type },
      read_only: -> (principal) { principal.read_only },
      uid: -> (principal) { principal.id }
  }

  config.organization_name = "Example"
  config.base_saml_location = "#{base}/saml"
  config.attribute_service_location = "#{base}/saml/attributes"
  config.single_service_post_location = "#{base}/saml/auth"
  config.session_expiry = 0
  config.algorithm = :sha256

  config.attributes = {
    "User id" => {
      "name" => "urn:oid:0.9.2342.19200300.100.1.1",
      "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
      "getter" => ->(principal) {
        principal.id
      },
    }
    "Email address" => {
      "name" => "email",
      "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
      "getter" => ->(principal) {
        principal.email
      },
    },
    "Name" => {
      "name" => "name",
      "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
      "getter" => ->(principal) {
        principal.name
      }
    },
    "Department" => {
      "name" => "department",
      "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
      "getter" => ->(principal) {
        principal.department
      }
    },
     "Position" => {
      "name" => "position",
      "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
      "getter" => ->(principal) {
        principal.position
      }
    },
     "User Type" => {
       "name" => "user_type",
       "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
       "getter" => ->(principal) {
         principal.user_type
       }
     },
     "Read Only" => {
       "name" => "read_only",
       "name_format" => "urn:oasis:names:tc:SAML:2.0:attrname-format:basic",
       "getter" => ->(principal) {
         principal.read_only
       }
     }

  }

  service_providers = {
    "serviceprovider.example.com" => {
      fingerprint: "<fingerprint of certificate>",
      metadata_url: "https://serviceprovider.example.com/users/saml/metadata",
      response_hosts: ["serviceprovider.example.com"]
    },
 }

  # `identifier` is the entity_id or issuer of the Service Provider,
  # settings is an IncomingMetadata object which has a to_h method that needs to be persisted
  config.service_provider.metadata_persister = ->(identifier, settings) {
    fname = identifier.to_s.gsub(/\/|:/,"_")
    `mkdir -p #{Rails.root.join("cache/saml/metadata")}`
    File.open Rails.root.join("cache/saml/metadata/#{fname}"), "r+b" do |f|
      Marshal.dump settings.to_h, f
    end
  }

  # `identifier` is the entity_id or issuer of the Service Provider,
  # `service_provider` is a ServiceProvider object. Based on the `identifier` or the
  # `service_provider` you should return the settings.to_h from above
  config.service_provider.persisted_metadata_getter = ->(identifier, service_provider){
    fname = identifier.to_s.gsub(/\/|:/,"_")
    `mkdir -p #{Rails.root.join("cache/saml/metadata")}`
    full_filename = Rails.root.join("cache/saml/metadata/#{fname}")
    if File.file?(full_filename)
      File.open full_filename, "rb" do |f|
        Marshal.load f
      end
    end
  }

  # Find ServiceProvider metadata_url and fingerprint based on our settings
  config.service_provider.finder = ->(issuer_or_entity_id) do
   service_providers[issuer_or_entity_id]
  end
end
```

Voila. we now have a working Ruby on Rails + devise SAML IDP application. We will continue to create a service provider application that will connect to the current SAML IDP.
