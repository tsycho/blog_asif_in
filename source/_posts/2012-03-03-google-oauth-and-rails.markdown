---
layout: post
title: "Google OAuth and Rails"
date: 2012-03-03 21:04
comments: true
categories: [oauth, rails, gmail, technical]
---

While building [InboxSlasher](http://www.inboxslasher.com), I needed to enable users to give me access to their Gmail accounts via OAuth. The [gmail_xoauth](http://github.com/nfo/gmail_xoauth) gem helped me do what I needed once the authentication was set up, but referred to [Google's python code](http://code.google.com/p/google-mail-xoauth-tools/wiki/XoauthDotPyRunThrough) for generating the actual OAuth tokens. Since I obviously couldn't use that from within my Rails app, I translated the Python code to Ruby as shown below.

Note: The Readme on Github for [gmail_xoauth](http://github.com/nfo/gmail_xoauth) now has a link to some Sinatra code for the OAuth token generation. I haven't checked it out. However, anyone implementing my solution should probably do so. Also, please let me know if that is an easier or better solution, and I will update this post.

Here's the code for the OAuth token generator. This is also saved as a [gist](https://gist.github.com/1970639).
<script src="https://gist.github.com/1970639.js?file=ruby_oauth_token_generator.rb"></script>

### How to use this
1) Save the above into a file that is accessible to your Rails controllers.


2) From the controller where you want to initiate the token generation request (UsersController in my case), call the generate_request_token() method. Save the oauth_token and oauth_token_secret from above into the User's model, and redirect the user to oauth_request_url.
``` ruby
oauth_request_url, oauth_token, oauth_token_secret = generate_request_token()
# save oauth_token and oauth_token_secret to @user
redirect_to oauth_request_url
```
3) Once the user has given your app permission (or refused to do so), Google will send a POST to the callback url that was specified above (see line #56). Modify the following code appropriately to handle the callback.
``` ruby
class OauthController < ApplicationController

	def authenticate
		oauth_token = params[:oauth_token]
		oauth_verifier = params[:oauth_verifier]
		
		@user = User.find_by_oauth_token(oauth_token)
		if !@user
			# Do something appropriate, such as a 404
		else
			begin
				oauth_tokens = get_access_token(oauth_token, @user.oauth_token_secret, oauth_verifier)
				# Update @user, save oauth_tokens in the database (in a secure way)
			rescue Exception => e
				# Something went wrong, or user did not give you permissions on Gmail
				# Do something appropriate, potentially try again?
				flash[:error] = "There was an error while authenticating with Gmail. Please try again."
			end			
			redirect_to root_url
		end
	end
	  
end
```

If everything above worked fine, you should now have the user's oauth_token and oauth_token_secret for Gmail. Use the [gmail_xoauth](http://github.com/nfo/gmail_xoauth) gem for the rest of your work.

### Notes:
1. I used 'anonymous' for the consumer token and secret. If you have a consumer token and secret from Google, you should use that instead.
2. If you need access to more Google services and not just Gmail, add them to params['scope'] on line #58.
3. I had to define my own method for URL escaping since CGI.escape was converting spaces into '+' instead of '%20', which was causing me problems. I am not sure what is the right approach here.

