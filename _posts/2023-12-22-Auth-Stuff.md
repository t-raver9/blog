---
layout: post
title:  "Auth Learnings"
---

I've hated working on anything auth-related pretty much the entirety of my short software career. I think it's because whenever I need to touch it I haven't built it, and it's such
chaos to figure out what's happening when working on it, and a nightmare to debug, and just, I dunno. It hasn't rubbed me the right way.

When a friend and I tried to start a platform earlier this year, the first thing I needed was auth, and I didn't have much of a better experience. You can see the
[front-end](https://github.com/t-raver9/pamper_client) and [back-end](https://github.com/t-raver9/pamper_server) I built here. I don't know how I got this to work, but it did;
I pulled the servers down so you'll have to take my word for it, but you could register and sign in and it would remember who you were. This was an utter mess, though. I didn't
want to refactor it because I didn't understand what I'd built, I'd just smashed together some swiss cheese security system using a bunch of blogs I found. My 97 year old grandmother
probably could have hacked into it.

Anyway, I started playing around with [AdonisJS](https://adonisjs.com/) earlier this week and decided to spin up a little toy project with the goal of understanding how the auth
worked, once and for all. The "Opaque Access Token" (OAT) method comes out-of-the-box with Adonis API applications, so I've been working with it for a couple of days and I think I've got
it down pat.

### Step 1: Create a User
This is exactly what it sounds like. You have a regular, old-fashioned CRUD endpoint that allows a user to register. This persists them in your DB with a password and some sort of
identifier. For this app, it's an email.

```
public async register({ request, response }: HttpContextContract) {
    try {
      const payload = await request.validate(RegisterValidator) # Payload has email + password
      const user = await User.create(payload)

      return response.status(200).json(user)
    } catch (e: any) {
      return response.status(500).json({
        message: 'Registration failed',
        error: e.message,
      })
    }
  }
```
Usually this would involve some token-y goodness, but for the sake of this post, lets say it doesn't.

### Step 2: Login
A user can now login using their email and password. This will do two things:
1) go to the database and check the email + password combo is valid, i.e. it matches those in step 1. And if so,
2) generate an Opaque Access Token (OAT)

But what _is_ this OAT? Can I microwave it for 90 seconds, add some berries and protein powder, and enjoy a low-GI high-nutrient breakfast? I'm afraid this isn't the case. This
OAT is nothing but a randomly generated string that gets stored in Redis, which is basically just a key-value database, alongside the user ID. Using the Redis CLI, I can see tokens
in there of this shape:

```
{
  "user_id": 1,
  "name": "Opaque Access Token",
  "token": "eb13033089e8f6c804e66afab89ee452577ceeecd5847e3003a1c9d806064ad5\"
}
```

With the OAT safe and sound in Redis, the server then sends it back to the client. So your Adonis login controller method might look something like this:
```
public async login({ request, response, auth }: HttpContextContract) {
    try {
      const { email, password } = await request.validate(LoginValidator)
      const token = await auth.use('api').attempt(email, password)

      return response.status(200).json(token)
    } catch (e: any) {
      return response.status(500).json({
        message: 'Login failed',
        error: e.message,
      })
    }
  }
```

And the response to your client looks like this:
```
{
	"type": "bearer",
	"token": "Y2xxZ2g5djI5MDAwMG00MWsyejg5Y2JsZA.wJUNchyhXuBP4PIHzFZsGbodpqQouekow6GwC35yCHzYqIzp9XqUTVmUhwks"
}
```

### Step 3: Subsequent Requests
Now that the client has the OAT, they need to include it in any requests they make to the server so the server knows they're an authenticated user. When the server receives the
request it will go to Redis and check if that token exists. If it does, congratulations, you've gained access to the holy land. Since Redis also holds the user_id, it can use
this to determine which parts of the application the user is authorized for, too.

### Shit diagram that shows what I just said
<img width="488" alt="image" src="https://github.com/t-raver9/blog/assets/33503958/23f45620-aa58-4c58-9a3a-df936dbb1c2f">

### My Takeaways and Learnings from this Escapade
This really isn't that hard. I wish I took a day to learn this like two years ago; if I had, I probably wouldn't have cringed so much whenever auth discussions came up. There's still other
auth methods I'd like to know better (JWT, basic, OAuth) so I'll play around with them in the coming months, too.
Generally though, I think this is an example of one of those things in tech where you can't really just dive into the code and figure it out along the way. This is how I operated
earlier in my career when I was literally just trying to code, but as you move into higher abstractions you need to read the docs and play around with toy examples until you
grok it.


