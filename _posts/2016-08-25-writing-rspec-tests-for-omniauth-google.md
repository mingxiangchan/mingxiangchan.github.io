---
layout: post
title: Writing Rspec Tests for Omniauth Google
tags: [ruby, rspec, rails, omniauth]
---

Recently I learnt a few things when setting up some Rspec tests to integrate Google Omniauth into a Rails app. This is a summary of my process while working on it.

Following BDD principles, I wanted to test the following:

1. Feature: User Experience using Omniauth to Log In
2. Controller: Receiving oauth details from oauth provider
3. Model: User Creation/Update in DB
<!--break-->
---

---

## Feature: User Experience using Omniauth to Log In
I wrote a simple Capybara spec to simulate users logging in with their Google account.

```ruby
RSpec.feature "UserCanLoginUsingGoogle", type: :feature do
  let(:mock_username){ "placheholder" }

  scenario 'user can log in using google' do
    visit sign_in_path
    click_link "Sign In Using Google"
    expect(current_path).to eq root_path
    expect(page).to have_content("Welcome, #{mock_username}")
  end
end
```

Those familiar with Omniauth and feature testing will immediately realize an issue with this process. Omniauth redirects users to the provider's website, allowing them to perform the login and authorization procedures without any involvement from our server. Our server will then wait for the provider to send some data related to the user's account and process it once it is received.

As such, we can not and should not simulate provider's server, e.g. `Choose which Google account you want to you`, `Do you allow this app to gain so and so permissions`. That process occurs without our server's knowledge. For our testing, we only need to simulate a possible response from the provider.

Fortunately, Omniauth already provides support for this with a test mode.

### Setup Omniauth Test Mode
Add this block in your test helpers. I added in in my `spec_helper.rb`, however you could also add it directly in a controller test or in a support file. Just make sure this code is ran before you run your tests.

```ruby
OmniAuth.config.test_mode = true

OmniAuth.config.mock_auth[:google_oauth2] = OmniAuth::AuthHash.new({
  :provider => 'google_oauth2',
  :uid => '123545',
  :info => {
    :name => "google_user",
    :email => 'test@gmail.com',
    },
  :credentials => {
    :token => '12345678'
  }
})
```
This `mock_auth` can be configured with whichever provider you are using e.g. `mock_auth[:facebook]`. You would structure this auth hash to simulate a response from the oauth provider. Also, we use the `OmniAuth::AuthHash.new` object as opposed to a normal hash so you can access the object using both method names and hash keys.

e.g.

```ruby
OmniAuth.config.mock_auth[:google_oauth2].info.name # => "google_user"
OmniAuth.config.mock_auth[:google_oauth2][:info][:name] # => "google_user"
```

The only other changes we would need to make to the capybara spec after this would be to update the name of the mocked user logging in using oauth with out mock object.

```ruby
  let(:mock_username){ OmniAuth.config.mock_auth[:google_oauth2].info.name }
```

---

## Controller: Receiving oauth details from oauth provider


```
Controller: Receiving oauth details from oauth provider
  if controller receive valid credentials from Oauth callback
    if user does not have an existing account
      user account is created in DB
    user oauth details are updated
    user is logged in
    user is redirected to home page
    welcome message is rendered
  if controller receives invalid credentials from Oauth callback
    user is redirected to login page
    error message is rendered
```

Most of the requirements are just standard controller tests. However, I learnt two new things when setting this up.

### 1. Request setting in Rspec
The omniauth provider information is received in the controller in `request.env["omniauth.auth"]`. A lot of guides would do something like `User.create_from_oauth(request.env["omniauth.auth"])` to create a user from this data.

This poses a bit of challenge since we usually just pass in data to controllers in rspec via parameters, e.g. `post :create, user: { name: "hello" }`

A bit of digging around and I found out that we could customize the `request` object before triggering the controller action in controller specs. So the only coded needed was as simple as:

```ruby
  before(:each) do
    auth_hash = OmniAuth.config.mock_auth[:google_oauth2]
    request.env["omniauth.auth"] = auth_hash
  end
```

### 2. Triggering controller actions in Rspec
I had created my callback route using the usual format from other guides.

```
get "/auth/:provider/callback" => "sessions#create_from_omniauth"
```

However I got stuck for a while on trying to trigger the `:create_from_omniauth` action in my controller specs. `get :create_from_omniauth` didn't work, raising a `Route not found error` instead.

Its a bit embarassing but I forgot that paramaters need to be passed even in `get` requests. Been testing the standard rails controller actions only so this is the first time I had to do it for a custom route. The solution was to use the following action:

```ruby
get :create_from_omniauth, provider: :google_oauth2
```

---

## Model: User Creation/Update in DB

```
Model: User Creation/Update in DB
  if user does not have an existing account
    a new account will be created for the user
    the oauth provider, uid, and token will be stored for the user
  if user has an existing account
    new account will not be created for the user
    the user's oauth token and uid for the oauth provider will be updated
```

This is the most straightforward test since we already know how to retrieve the mock `OmniAuth::AuthHash`, so we would just use it in our tests.

```ruby
  let(:auth_hash){ OmniAuth.config.mock_auth[:facebook] }
  let(:user){ User.find_or_create_from_omniauth(auth_hash) }
````

The rest are the normal rspec tests like:

```ruby
it 'should update user with omniauth provider details' do
  expect(user.provider).to eq auth_hash.provider
  expect(user.uid).to eq auth_hash.uid
  expect(user.token).to eq auth_hash.credentials.token
end
```

And thats it. Happy Testing!

