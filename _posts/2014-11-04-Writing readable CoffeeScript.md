Hello! At the time this article was written, CoffeeScript was a great alternative to vanilla JS and I would have wholeheartedly recommended it for new projects. Since then, most of the cool stuff from CoffeeScript (plus a lot more) has been brought into mainline JS with ES6 and ES7. If you aren't already, I'd recommend using them over CoffeeScript nowadays.

---

In the past couple months, I've started to writing most of my JavaScript code in CoffeeScript. CoffeeScript is easier to write than JS because it's less verbose and it gives you tools to be more concise, but sometimes the language's focus on brevity makes reading it more difficult. For that reason I'd like to lay out some readability-aiding patterns and practices that I've found useful, with the hope of helping other devs write more clear and maintainable code.


###Don't omit parentheses just because you can
Here's a straightforward example from a codebase I worked on (with names changed).
```coffeescript
	queue.push db.doSomeAction user.id, req.params.arg1, req.params.arg2
```
I certainly didn't notice the missing comma between `db.doSomeAction` and `user.id` the first time I saw code like this floating around, but the difference is huge. Adding parenthesis makes it clear that we're pushing *the return value of the function db.doSomeAction* into the queue:
```coffeescript
	queue.push db.doSomeAction(user.id, req.params.arg1, req.params.arg2)
```
Notice I didn't add parenthesis for the call to `queue.push`. There, the parens would add clutter without aiding meaning, so omission is preferred.

###Implicit return can be too clever

CoffeeScript automatically returns the last statement in a function for you, but being explicit aids readability, especially for JS developers who aren't used to that pattern. If I'm writing a function with the goal of using its return value, I will generally write out the `return`. Coming back to my code later, I've found it easier to understand what my intentions were, especially with more complicated methods where the value returned is the direct result of some inner function call.

###@ notation isn't right for every situation
CoffeeScript aliases `this` to `@`, giving a Ruby-like syntax for property and method access. I love it for accessing methods and instance properties, but if you need to pass 'this' as an argument, you should type it out. `doSomething(@)` just doesn't read, nor does `return @`.

###Write functions that take named parameters
You see a lot of method calls in CoffeeScript (and JavaScript) that look like this:
```coffeescript
	person.eat 'applesauce', 'spoon', true, 100, true, -> ...etc...
```
It's easy to guess at what the arguments are doing for the first two, but what about the rest? Named parameters fix this. With a little adjustment to the definition, CoffeeScript will let you call that same method like this:
```coffeescript
    person.eat 'applesauce', with: 'spoon', warm: 'true', duration: 80, washBowl: true, finished: -> ...etc...
```
Way more readable. Here's how to write the definition:
```coffeescript
	person.eat = (dish, {with, warm, duration, washBowl, finished}) ->
      # CoffeeScript takes care of variable binding:
      console.log "eating #{dish} with #{with} for #{duration}"
      finished()
```    
Sometimes it helps to name parameters differently for external and internal access. In the example above, the `with` parameter name reads well in the call, but not when it's used inside the method itself. You can fix that like this:
```coffeescript
	person.eat = (dish, {with: utensil, warm, duration, washBowl, finished}) ->
      # now you pass the parameter using 'with' like before, and use it as 'utensil':
      console.log "eating #{dish} with #{utensil} for #{duration}"
```
One important thing to be aware of is that if *all* of the named parameters are left out of a call, the method will throw an error. The fix, if you want to make all your named parameters optional, is to provide a default value:
```coffeescript
	person.walk = (location, {speed, shoes, route} = {}) ->
```  
###Set instance variables directly in the arguments list
CoffeeScript has a nice syntax for setting instance variables that are passed as arguments:
```coffeescript
    constructor: (@type, {@size, x, y}) ->
      # now there's no need to write: 
      # @type = type 
      # @size = size
      # etc
```              
This is a nice way to avoid some boilerplate when you want to set the value of some argument as a property of the instance.

###Use optional chaining to avoid boilerplate undefined-checking

When accessing properties that might not exist deep in an object, we're used to checking that each level is there, one by one, to avoid a TypeError.
```coffeescript
	if result and result.bike and result.bike.wheels
	    tires = result.bike.wheels.tires
```
CoffeeScript lets us do this instead:
```coffeescript
	tires = result.bike?.wheels?.tires
 ```
