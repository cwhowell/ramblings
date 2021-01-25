# Monoliths
A 'microservice based app' is the opposite of a 'monolithic app'. You can think of a monolithic app as one were all of the functionality is part of the same program. So if you're developing an eCommerce site, you might have different functions:
* A function that displays a list of items in a category. This is what's used to generate a "View Products" page.
* A function that displays information about a single product. This is what's used to generate a "View Product Details" page.
* A function adds a product to a user's cart.
* We want to give the user an indication of availability, so we need a function to check and update the stock-level of items.
* A function that let's a user checkout to buy their products.
* We'd like the user to come back so we let them create an account.
* So now we need functions to power log-in and log-out.
* We want to recommend products to a user based on their history so we need a function for that.
* We want our staff to be able to add and edit products.
* When we add products we want to add pictures of them, and maybe they come from different sources but we want them all to be the same size, so we resize the pictures automatically.

When you build a monolith, all of the code to implement these features is written into a single program. You then run that program by building it, and putting it somewhere along with its dependencies. Dependencies can be other programs, or pieces of other people's code (that on their own aren't quite programs, but when you mix them in with your code they help build your program). A dependency might be an authentication library that has functions for dealing with passwords (because it's a bad idea to write cryptographic code yourself), and if you're resizing images you'll probably need a library for that, because there's no point writing support for all the different image formats when someone else has already done that.

## Benefits
* Simplicity - there is one program, and it does everything that needs to be done
* Fewer points of failure - as we'll see below, microservices introduce communication dependencies that can be points of failure
* Easier to deploy - the architecture needed to deploy a monolith is conceptually much simpler: take a server, install the dependencies, install the program, and connect it to the internet. That's it.
* Easier to troubleshoot - because it's all one program, it's easy to inspect what it's doing at any given moment, using tools like a debugger or logging.

## Consequences
* As the feature set grows, the application becomes larger. Depending on the language used, it might take longer to compile. The larger an application is, the harder it can be for a new developer to get up to speed.
* Updating one feature requires updating the whole app (so you probably want to do it at like, 0300, when noone is shopping).
* If you're not careful, you might introduce a bug in one feature that effects other features.
* It has impacts on scaling

## Scaling
Big sale for the holidays? Many more users hitting the site than normal? You'll probably need to scale. That can take a couple of forms:
* Vertical scaling: Your app usually runs on a server with 24 cores and 64GB of RAM. You need to process more requests per second (cause there are lots of users!) so instead you run it on a server with 96 cores and 128GB RAM.
* Horizontal scaling: You usually run one instance of your app. To process more requests, you instead run four instances, and distribute incoming customer requests to be handled by each of them.

Traditionally, scaling was mostly vertical - the company would buy a bigger machine. But this has a problem: you can only vertically scale so far. At a certain point, you reach the maximum number of cores and RAM that can fit in a single machine (though you'll probably go broke first). Google and other big tech companies realised that instead they could scale horizontally, buying many cheaper machines and running multiple instances of their code. But horizontally scaling a monolith means you're scaling all of the monolith. Every new instance needs to have all the code and dependencies, even if parts like the image manipulation functions aren't being used that much - the number of staff and the rate they add products is probably the same regardless of how big your sale day is with customers. 

# Microservices
The key to microservices lies in segmentation - instead of building everything into one big program, build lots of smaller programs and then make them talk to each other. So to replicate the feature set above, you might have:
* A products service, that has these features:
    * Add product
    * Edit product
    * List products
    * Retrieve product details
* An inventory service, that keeps track of how much stock is available
* An accounts / authentication service, that handles:
    * Creating user accounts
    * Sign in and sign out
* A cart service, that tracks what's in a user's cart and handles adding and removing things
    * You can't checkout with at least one product so maybe checkout is just a part of the cart service
* A recommendations service - this looks at a user's history and generates the recommendations

## Making them talk to each other
In the monolith, everything is running as part of the same program, so communication between features is trivial - just call a function. Microservices add a layer of complexity - the checkout service needs to talk to the product service so it knows how much each product costs, and when the customer buys something it needs to tell the inventory service so that its "amount in stock" number goes down. In practice, it's common for microservices to do this by exposing their own Application Programming Interfaces (APIs), often over HTTP - the same protocol that's used to power the web. This is a simple solution because web developers already know how to write APIs - a monolith can expose one too, and they already understand HTTP. There are some other options though:
* Remote Procedure Calls (RPCs). There are a variety of different protocols that can be used here - the key takeaway is that it's just another way of having one program ask another to do some work over a network.
* Queues and message buffers. Consider our photo-resizing feature. We don't actually need that resize to finish in order to add the product, as long as it gets done at some point. So when the staff member clicks "save", their image is uploaded and put into a queue. The image service watches that queue and takes work out of it whenever it's free. This way you can do a big bulk import of images and have them process away in the background.

## Benefits
* Enforces the separation of concerns - it can be easier to divide work among teams if each team is responsible for their own microservice. APIs can act like contracts - "As long as you tell me a price when I tell you a product, I don't care how you wrote the code or even what language it's in"
* Can be easier to deploy updates to one component - if marketing is trying a new recommendation algorithm they can update their microservice without updating the rest of the app.
    * In some architectures they can do this by deploying the new version of their service, then redirecting the rest of the app to the new version. Once the old version has finished responding to its last request, it can be destroyed.
    * In less complicated architectures they may just destroy the old version and then deploy the new one, and in the few seconds between versions the site may show users a set of default recommended products, instead of their personalised offers.
* You can scale only the services that need it. In the example above, we might horizontally scale the cart/checkout service, and put caching (see below) in front of the products service. The rest doesn't need to scale, so scaling is simpler and easier (don't have to make sure the image processing library is installed on all the app servers if they're only running the checkout service).
    * Caching: The products service looks up a product in the database and returns information about it. Once added, this info doesn't usually change very much. Rather than asking the database every time, we can cache the info in a place that's faster to read from, speeding up response times when customers visit.
        * To add caching to our monolith we'd need to write code, but if the products microservice is using an HTTP-based API we could just slot an existing HTTP caching product in the middle - the caching product doesn't care that it's caching product info, because it all just looks like HTTP traffic.
* If well separated, a bug that causes the authentication/accounts microservice to crash won't impact the rest of the application. If the checkout service allows people to checkout as guests, then an accounts crash might not even impact sales too badly, whereas in a monolith that crash might have caused the entire application (and thus, the entire website) to go down.


## Consequences
* More points of failure. Before your application depended on one program on one server (before scaling). Now there are multiple programs on multiple servers with some kind of network communication in between.
* Harder to troubleshoot - you can't view the entire app's state in one debugger. You can make sure all your microservices send out logs but now you have to collect, correlate, and display them (there are entire companies that make a bunch of money just doing this for software teams).
* Harder to understand the system as a whole - it might be faster for a new dev to learn their piece of the puzzle, but how it fits in to the whole may be harder to reason about.

# Which is better?
Surprise surprise, there is no right answer to this. Like many things in tech, microservices have caught on in part because it's what big companies like Google and Facebook started doing. For their needs, the benefits of microservices can often outweigh the consequences. But the scale of Google is not the scale of even the largest eCommerce site (Google calls it "Planet Scale" for a reason). Plenty of companies use microservices for things that will never grow to a size that would require them, and as a result they expend a lot of effort on building the architecture needed to run all their services (this is fun for a nerd like me, but often not a good use of time and money). Microservices have been around long enough that I've observed some amount of bounce-back - i.e. people writing posts about how actually, monoliths aren't that bad. But I don't think they've been around long enough for us to know for sure if they're a fantastic or terrible idea. They'll likely end up somewhere in the middle, like most things.