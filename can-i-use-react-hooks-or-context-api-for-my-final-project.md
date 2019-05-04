# DRAFT - THIS ANSWER IS CURRENTLY IN BETA :stuck_out_tongue_winking_eye:
### CAN I USE HOOKS OR CONTEXT API FOR MY FINAL PROJECT?  DOES EITHER OF THOSE REPLACE REDUX?

The short answers are no, no, and not exactly.  But please do read the long answer!

_Of course_ you should look into hooks and implement them in upcoming React apps!  And _of course_ you should check out the Context API and see how it feels!!!  Of course you should stay up-to-date on the libraries you are putting on your resumes to get jobs!!  

_But_ what is in the curriculum now is still highly relevant and useful.  Stick to what's there and learn one thing at a time.  Learning about Redux and class components is still an excellent way (I would argue still _the_ way, but that's just an opinion) to learn React!   Here is a very brief look at Redux, hooks, and Context API in contrast to what's in the curriculum now...

#### Redux

Dan Abramov is the author of Redux, and a member of React's core dev team.  He wrote this blog.  Go read it, if you like:

[_You Might Not Need Redux_ by Dan Abramov, the author of Redux](https://medium.com/@dan_abramov/you-might-not-need-redux-be46360cf367)

Now that you're angry for having learned and used Redux in a final project that's way too simple for it, settle down, take a breath, and consider _why_ we teach Redux.  As Dan points out, Redux is a particular application of a pattern that is not new -- a way to carefully prescribe the way to change state and keep track of those changes.  The benefits can be tremendous -- bug tracking, less mismatched-state issues, and a single source of truth for your app.  Learning a pattern like this is good for you, not just because you can now use Redux, but because when you get hired as a dev you now understand a pattern you may see in various forms, and you understand why it's useful.

#### React Hooks

[When hooks were released](https://www.youtube.com/watch?v=dpw9EHDh2bM), the React core team basically shouted "Do not go back and change everything over to hooks, that's completely unnecessary!  Just feel free to start using them if you like them!  Oh and give us feedback!" ... So, the vast majority of React code that's in production now is 'hook-less', and full of stateful class components, and some folks may prefer class components anyway, so that syntax is not going anywhere any time soon.  Also, learning the ins and outs of class components first will help you both understand and appreciate hooks that much more when you do learn them.  Hooks do _not_ replace Redux, and really don't have to do with Redux so much.  You can actually have a mix of stateful components and still use Redux, saving your Redux store for state that truly needs to be global and available across multiple components.  So Redux and hooks can play nice, as hooks just add state to functional components without the need for class components.  I would venture to say that Redux and Context make less sense together.

#### Context API

[Check it out](https://reactjs.org/docs/context.html).  Do not take this oversimplified analogy as gospel, but you can think of Context as super-lightweight Redux without as many rules, and without as many tools.  (I made that up, it's a work in progress.)  Using the Context API does allow us to access state down to ancestors without passing props through a bunch of intermediate children components.  So does Redux.  Redux, however, allows us powerful dev tools and a global state with tightly controlled rules for change.  Context, not so much.  This does not mean one is better than the other; they are simply different and good for different use cases.  Yes, your project is likely too small and uncomplicated to warrant using Redux.  Yes, Context could make more sense and involves less syntax.  Yes, we still ask that you learn and use Redux.  (Trying to come up with a decent analogy here.. )  It's kind of like, "can't I just learn use a push mower rather than a John Deere tractor?"  Well, yes, but the John Deere tractor has a lot more to offer.  Hmm.. I think this analogy needs work...

#### Conclusion, for now

Understand that this answer, in the context of this curriculum and these projects, is subject to change.  It _will_ change.  And then it will change again.  And again.  As you likely have realized by now, and indeed you better have realized by now, programming tools change, evolve, die out, and are born again every day.  If you memorized every library available for use for JS today, and mastered every function, method, and feature in existence, tomorrow you would have lots to learn as these things tend to be living, changing structures.  That's why learning _how to learn_ is as important, maybe more important, than the granular content of this curriculum.  Because it's all going to change sooner or later, and you better keep up!  Learn what's there, one step at a time.  Build your project to spec as it currently stands, then feel free to explore hooks and Context and MERN stacks and Elixir and anything else that strikes your fancy!     
