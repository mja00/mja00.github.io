---
layout: post
title: Building site wide bans using Phoenix with Elixir
subtitle: Let's just say it wasn't easy
gh-repo: glimesh/glimesh.tv
gh-badge: [star, fork]
tags: [elixir]
comments: false
---

So I've been spending the last few months working on an [up and coming](https://github.com/Glimesh/glimesh.tv) streaming platform. Obviously we were going to need a way to globally ban accounts from the site without removing them. This is no way the best way of doing it but it works for what we needed for an alpha launch. You can view the entire PR [here](https://github.com/Glimesh/glimesh.tv/pull/132).

---

## Prepping the user table for bans

First we needed to track who was actually banned, not really hard, just needed to edit our users schema to include a `is_banned` flag. Super simple, just needed to update the schema to:

```elixir
schema "users" do
    # Various more fields
    field :is_banned, :boolean, default: false
    field :ban_reason, :string
end
```
For our case we also needed to add the `is_banned` flag to the register changeset with:
```elixir
def registration_changeset(user, attrs) do
    user
    |> cast(attrs, [
        :username,
        :email,
        :password,
        :is_banned
    ])
end
```
With that you can generate a new migration, `mix ecto.gen.migration add_global_bans`, and we're ready to actually implement the bans into the site.

---

## Stopping banned users from logging in

This was also pretty simple. For Glimesh we have a login page for our users, we just needed to add a check for if the user is banned. This was all handled in our `user_session_controller.ex` file. 

We replaced our create function with
```elixir
  def create(conn, %{"user" => %{"email" => email, "password" => password} = user_params}) do
    if user = Accounts.get_user_by_email_and_password(email, password) do
      attempt_login(conn, user, user_params)
    else
      render(conn, "new.html", error_message: gettext("Invalid e-mail or password"))
    end
  end
```
And then from there we did some good ol' Elixir pattern matching on attempt_login
```elixir
# Attempts a login if the user isn't banned
def attempt_login(conn, %{is_banned: false} = user, user_params) do
    # Since we use 2FA we check to see if they have it enabled
    if user.tfa_token do
        # If they do, we want to render the view for entering the code.
        # Later was changed to a different page
        conn
        |> put_session(:tfa_user_id, user.id)
        |> render("tfa.html", error_message: nil)
    else
        # And if they don't have 2fa then we log them in
        UserAuth.log_in_user(conn, user, user_params)
    end
end

# Attempts a login if the user is banned
# With pattern matching we don't do any checks, this will only fire if the user is banned
def attempt_login(conn, %{is_banned: true} = user, user_params) do
    # Basically re-renders the login page with an error message
    render(conn, "new.html",
       error_message:
         gettext(
             "User account is banned. Please contact support at %{email} for more information.",
             email: "support@example.com"
         )
    )
end
```
This essentially stops any banned account from logging in. Now how did we handle already logged in users that get banned? Relatively simply. 

---

## Handling logged in users that get banned

For this I basically created an entire plug to check if the user is banned as they navigate the site, and if they get banned it invalidates their session(basically logging them out). The plug I created is somewhat simple, it pretty much is called on every page load, it will get the user from the connection and check if they're banned. If they are indeed banned it will trigger a ban user function in our user auth module(basically the same as logging out minus a redirect), and then replacing their path to our homepage(needed otherwise Cowboy gets **very** angry).

>plugs/ban_plug.ex

```elixir
defmodule GlimeshWeb.Plugs.Ban do
  import Plug.Conn

  alias GlimeshWeb.UserAuth

  def init(_opts), do: nil

  # This is called every time a new page is loaded(so it needed to be quick)
  def call(conn, _opts) do
    user = conn.assigns.current_user

    if is_user_banned(user) do
      conn
      |> UserAuth.ban_user()
      |> replace_path("/")
    else
      conn
    end
  end

  defp is_user_banned(user) do
    case user do
      nil -> false
      _ -> if user.is_banned, do: true, else: false
    end
  end

  defp replace_path(conn, path) do
    conn
    |> Map.replace!(:request_path, path)
    |> Map.replace!(:path_info, ["banned"])
  end
end
```

And inside of the [user_auth.ex](https://github.com/Glimesh/glimesh.tv/blob/dev/lib/glimesh_web/controllers/user_auth.ex#L91-L102) file I added a new function for when a user is banned.

```elixir
  def ban_user(conn) do
    user_token = get_session(conn, :user_token)
    user_token && Accounts.delete_session_token(user_token)

    if live_socket_id = get_session(conn, :live_socket_id) do
      Endpoint.broadcast(live_socket_id, "disconnect", %{})
    end

    conn
    |> renew_session()
    |> delete_resp_cookie(@remember_me_cookie)
  end
```
This here is what actually does the "logging out" of the user. You may notice the [`log_out_user/1`](https://github.com/Glimesh/glimesh.tv/blob/dev/lib/glimesh_web/controllers/user_auth.ex#L77-L89) function above it but the [`redirect/2`](https://hexdocs.pm/phoenix/Phoenix.Controller.html#redirect/2) at the end seems to break the funkiness we had to do for bans. Definitely not ideal but it works for now and we plan on revamping that part. 

---

## Finishing up the feature
After the schema was updated, logins properly handled banned users, and sessions were invalidated on ban, there were just a few small things to clean up here and there.

#### Making it so banned users couldn't chat
For this I just added a check to see if the user was banned and if so it raised an ArgumentError with a `User must not be banned` message. This was later swapped to our chat error handler once moderation tools were added to the chat. 

#### Adding an ignore_banned argument to our user lookup functions
Another simple task, since we wanted to hide any banned user from being searched I added `ignore_banned \\ false` to our username lookup functions that by default would have Ecto skip over any banned users. This argument will come back later when we're creating our admin team's dashboard and they need to be able to search for banned users. But pretty much everything else will ignore banned users. 

---

## All done! 

That's pretty much it for the site wide ban feature. We wanted to make sure it wasn't overly complex and could be reliable. It's most definitely not the best way, and I've already admitted that to the team, we'll be re-doing it at somepoint in the future but needed something that we could use during our alpha. This was also created on 09/30/2020, which was about 2 months into my journey learning Elixir. Again, if you wanna view the entire PR to see the code around this, you can do so [here](https://github.com/Glimesh/glimesh.tv/pull/132). 