# Agentic - LLM store
I wasn't originally going to make this writeup, but I will out of spite due to how long it took for me to complete this despite how easy it ended up being.

## Background Information
Unfortunately, the challenge infrastructure was instantly taken down during competition end, so i'll provide the background information verbally.
This writeup is going to be kind of bad because of the lack of images. I'll fill the images in once source code comes out.

We are given a simple store application with various products. There is no option to purchase any products, however there is functionality for comments on specific products, 
login/registration, and an AI bot running co-pilot.  

Upon querying the bot for what API it has access to, we are given:
**get_product_reviews_from_agent**
Querys a backend LLM agent to obtain the product and comments for a specific item. The backend bot will parse a backend database storing the products, descriptions, and comments.
Any information identified will be returned to the front-end copilot bot, which is forwarded to the user.

**get_APIKey**
Returns the current logged in user's API key. Bot was specifically instructed not to run this API.

**get_product_from_description**
Given a user's input, attempts to locate a specific product following that description.


