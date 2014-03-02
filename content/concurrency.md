---
title: Concurrency & Parallelism
link_title: Concurrency & Parallelism
kind: documentation
draft: true
---

One of Clojure's big selling points is its support for concurrent
programming. In this chapter, you'll learn the following:

* Intro to concurrency & parallelism
    * Definitions
    * Why should you care?
* Easy concurrent ops
* What are the dangers?
* The Clojure model of state
* Tool 1: Statelessness
* Tool 2: Atomic updates
* Tool 3: STM
* Tool 4: core.async

## Who Cares?

Why should you bother with any of this? The reason is that parallel
programming is essential for writing performant applications on modern
hardware. In recent years CPU designers have focused more on enabling
parallel execution than on increasing clock speed. This is mostly
because increasing clock speed requires exponentially more power,
making it impractical. You can google "cpu power wall", "cpu memory
wall" and "instruction-level parallelism wall" to find out more. As
those creepy Intel bunnies continue smooshing more and more cores
together, it's up to you and me to squueze performance out of them.

Performance can be measured as both **latency**, or the total amount of
time it takes to complete a task, and **throughput**, or the number of
tasks a program completes per second. Parallelism can improve both, as
this diagram shows:

TODO: diagram serial vs parallel execution of tasks

## Concurrency and Parallelism Defined

Concurrent and parallel programming involves a lot of messy details at
all levels of program execution, from the hardware to the operating
system to programming language libraries to the code that you're
typing out with your very own fingers. In this section I'll ignore
those details and instead focus on the implementation-independent
high-level concepts.

### Managing Tasks vs. Executing Tasks Simultaneously

**Concurrency** refers to *managing* more than one task at the same
time. "Task" just means "something that needs to get done." We can
illustrate concurrency with the song "Telephone" by Lady Gaga. Gaga
sings,

    I cannot text you with a drink in my hand, eh

Here, she's explaining that she can only manage one task (drinking).
She flat out rejects the suggestion that she manage more than one
task. However, if she decided to process tasks concurrently, she would
sing,

    I will put down this drink to text you,
    then put my phone away and continue drinking, eh

In this hypothetical universe, Lady Gaga is *managing* two tasks:
drinking and texting. However, she is not *executing* both tasks at
the same time. Instead, she's switching between the two. This is
called **interleaving**. Note that, when talking about interleaving,
you don't have to fully complete a task before switching; Gaga could
type one word, put her phone down, pick up her drink and have a sip,
then switch back to her phone and type another word.

**Parallelism** refers to *executing* more than one task at the same
time. If Madame Gaga were to execute her two tasks in parallel, she
would sing:

    I can text you with one hand
    while I use the other to drink, eh

Clojure has many features which allow you to easily achieve
parallelism. Whereas the Lady Gaga system achieves parallelism by
simultaneously executing tasks on multiple hands, computer systems
generally achieve parallelism by simultaneously executing tasks on
multiple cores or processors.

You can see from this definition that parallelism is a subclass of
concurrency: in order to execute multiple tasks simultaneously, you
first have to manage multiple tasks. Concurrency can be seen as
potential paralellism.

It's important to distinguish parallelism from **distribution**.
Distributed computing is a specialization of parallel computing where
the processors don't reside in the same computer and where tasks are
distributed to computers over a network. It'd be like Lady
Gaga asking Beyoncé, "Please text this guy while I drink."

I'm going to use "parallel" only to refer to cohabitating processors.
While there are Clojure libraries that aid distributed programming,
this book only covers parallel programming. That said, the concepts
you'll learn apply directly to distributed programming.

### Blocking and Async

One of the big use cases for concurrent programming is for
**blocking** operations. "Blocking" really just means waiting and
you'll most often hear it used in relation to I/O operations. Let's
examine this using the Concurrent Lady Gaga example.

If Lady Gaga texts her interlocutor and then stands there with her
phone in her hand, staring at the screen for a response, then you
could say that the "read next text message" operation is blocking. If,
instead, she tucks her phone away so that she can drink until it
alerts her by beeping or vibrating, then you could say she's handling
the "read next text message" operation **asynchronously**.


### Concurrent Programming, Parallel Programming

"Concurrent programming" and "parallel programming" refer to how you
decompose a task into parallelizable sub-tasks and the techniques you
use to manage the kinds of risks that arise when your program executes
more than one task at the same time.

To better understand those risks and how Clojure help you avoid them,
let's examine how concurrency and parallelism are implemented in
Clojure.

## Clojure Implementation: JVM Threads

I've been using the term "task" to refer to *logical* sequences of
related operations. Texting consists of a series of related operations
separate from those involved in pouring a drink into your face, for
example. So, "task" is an abstract term which says nothing about
implementation.

In Clojure, you can think of your normal, serial code as a sequence of
tasks. You indicate that tasks can be performed concurrently by
placing them on **threads**.

### So What's a Thread?

I'm glad you asked! Rather than giving a technical definition of a
thread, though, I'm going to describe a useful mental model of how
threads work.

I think of a thread as an actual, physical piece of thread that's been
threaded through a sequence of instructions. In my mind, the
instructions are marshmallows, because marshmallows are delicious. The
processor executes these instructions in order. I envision this as an
alligator consuming the instructions, because alligators love
marshmallows (true fact!). So executing a program looks like a bunch
of marshmallows strung out on a line with a pacifist alligator
traveling down the line and eating them one by one.

Here's a single-core processor executing a single-threaded program:

TODO: add image here

A thread can "spawn" a new thread. In a single-processor system, the
processor switches back and forth between the threads (interleaving).
Here's where we things start to get tricky. While the processor will
execute the instructions on each thread in order, it makes no
guarantees about when it will switch back and forth between threads.

Here's an illustration of two threads, "A" and "B", along with a
timeline of how their instructions could be executed. I've shaded the
instructions on thread B to help distinguish them from the
instructions on thread A.

TODO: add image here

Note that this is just a *possible* ordering of instruction execution.
The processor could also have executed the instructions in the order,
"A1, A2, A3, B1, A4, B2, B3", for example. The main idea is that you
can't know what order the instructions will actually take. This makes
the program **nondeterministic**. You can't know beforehand what the
result will be because you can't know the execution order.

Whereas the above example showed concurrent execution on a single
processor through interleaving, a multi-core system will assign a
thread to each core. This allows the computer to execute more than one
thread simultaneously. Each core executes its thread's instructions in
order:

TODO: image

As with interleaving on a single core, there are no order guarantees
so the program is nondeterministic. To make things even more fun, your
programs will typically have more threads than cores, so each core
will likely perform interleaving on multiple threads.

### The Three Goblins: Reference Cells, Mutual Exclusion, Dwarven Berserkers

To drive this point home, imagine the program in the image above
includes the following pseudo instructions:

| ID | Instruction |
|----+-------------|
| A1 | WRITE X = 0 |
| A2 | READ X      |
| A3 | PRINT X     |
| B1 | WRITE X = 5 |

If the processor follows the order "A1, A2, A3, B1" then the program
will print the number `0`. If it follows the order "A1, B1, A2, A3",
then the program will print the number `5`. Even weirder, if the order
is "A1, A2, B1, A3" then the program will print `0` even though `X`'s
value is `5`.

This little thought experiment demonstrates one of the three central
challenges in concurrent programming, a.k.a "The Three Concurrency
Goblins." We'll call this the "reference cell" problem. There are two
other problems involved in concurrent programming: mutual exclusion
and the dwarven berserkers problem.

For mutual exclusion, imagine two threads are each trying to write the
text of a different spell to a file. Without any way to claim
exclusive write access to the file, the spell will end up garbled as
the write instructions get interleaved. These two spells:

    By the power invested in me
    by the state of California
    I now pronounce you man and wife

    Thunder, lightning, wind and rain,
    a delicious sandwich, I summon again

Could get written as this:

    By the power invested in me
    by Thunder, lightning, wind and rain,
    the state of California
    I now pronounce you a delicious man sandwich, and wife
    I summon again

Finally, the dwarven berserker problem can be stated as follows.
Imagine four berserkers are sitting around a rough-hewn, circular
wooden table and comforting each other. "I know I'm distant toward my
children, but I just don't know how to communicate with them," one
growls. The rest sip their coffees and nod knowingly, care lines
creasing their eyeplaces.

Now, as everyone knows, the dwarven berserker ritual for ending a
comforting coffee klatch is to pick up their "comfort sticks"
("double-bladed waraxes") and scratch each others' backs. One waraxe
is placed between each pair of dwarves, like so:

Their rituaal proceeds thusly:

1. Pick up the *left* waraxe, if available
2. Pick up the *right* waraxe, if available
3. Comfort your neighbor with vigorous swings of your "comfort sticks"
4. Release both waraxes
5. Go to 1

Following this ritual, it's entirely possible that all dwarven
berserkers will pick up their left comfort stick and then block
indefinitely waiting for the comfort stick to their right to become
available, resulting in **deadlock**.

The takeaway here is that concurrent programming has the potential to
be confusing and terrifying. But! With the right tools, it's
manageable and even fun. Let's start looking at the right tools.

## Futures, Delays, and Promises

Futures, delays, and promises are easy, lightweight tools for
concurrent programming. In this section, you'll learn how each works
and how to use them together to defend against the Reference Cell
Concurrency Goblin and the Mutual Exclusion Concurrency Goblin. You'll
discover that, while simple, these tools can go a long way toward
meeting your concurrency needs.

### Futures

In Clojure, you can use **futures** to place a task on another thread.
You can create a future with the `future` macro. Try this in a REPL:

```clojure
(future (Thread/sleep 4000)
        (println "I'll print after 4 seconds"))
(println "I'll print immediately")
```

`Thread/sleep` tells the current thread to just sit on its ass and do
nothing for the specified number of milliseconds. Normally, if you
evaluated `Thread/sleep` in your REPL you wouldn't be able to
evaluate any other statements until the REPL was done sleeping; your
REPL would be blocked. However, `future` throws your call to
`Thread/sleep` into another thread, allowing the REPL's thread to
continue, unblocked.

Futures differ from the values you're used to, like hashes and maps,
in that you have to *dereference* them to obtain their value. The
value of a future is the value of the last expression evaluated. You
can dereference a future with either the `deref` function or the `@`
reader macro. Try this out to use them both:

```clojure
(let [result (future (println "this prints once")
                     (+ 1 1))]
  (println "deref: " (deref result))
  (println "@: " @result))
; =>
; "this prints once"
; deref: 2
; @: 2
```

Notice that the the string `"this prints once"` indeed only prints
once, showing that the future's body runs only once and the result
gets cached.

Dereferencing a future will block if the future hasn't finished
running:

```clojure
(let [result (future (Thread/sleep 3000)
                     (+ 1 1))]
  (println "The result is: " @result)
  (println "It will be at least 3 seconds before I print"))
; =>
; The result is:  2
; It will be at least 3 seconds before I print
```

Sometimes you want to place a time limit on how long to wait for a
future. To do that, you can pass the number of milliseconds to wait to
`deref` along with the value to return if the deref does time out. In
this example, you tell `deref` to return the value `5` if the future
doesn't return a value within 10 milliseconds.

```clojure
(deref (future (Thread/sleep 1000) 0) 10 5)
; => 5
```

Finally, you can interrogate a future to see if it's done running with
`realized?`:

```clojure
(realized? (future (Thread/sleep 1000)))
; => false

(let [f (future)]
  @f
  (realized? f))
; => true
```

Futures are a dead simple way to sprikle some concurrency on your
program. On their own, they give you the power to chuck tasks onto
other threads, allowing you to increase performance. They don't
protect you against The Three Concurrency Goblins, but you'll learn
some ways to protect yourself with delays and promises in just a
minute. First, though, let's take a closer look at dereferencing.

### Dereferencing

Dereferencing is easy to understand, but it implies some nuances that
aren't immediately obvious. It allows you more flexibility than is
possible with serial code. When you write serial code, you bind
together these three events:

* Task definition
* Task execution
* Requiring the task's result

As an example, take this hypothetical code, which defines a simple api
call task:

```clojure
(web-api/get :dwarven-beard-waxes)
```

As soon as Clojure encounters this task definition, it executes it. It
also requires the result *right now*, blocking until the api call
finishes.

Futures, however, allow you to separate the "require the result" event
from the other two. Calling `future` defines the task and indicates
that it should start being evaluated immediately. *But*, it also
indicates that you don't need the result immediately. Part of learning
concurrent programming is learning to identify when you've created
these chronological couplings when they aren't actually necessary.

When you dereference a future, you indicate that the result is
required *right now* and that evaluation should stop until the result
is obtained. You'll see how this can help you deal with the mutual
exclusion problem in just a little bit.

Alternatively, you can ignore the result. For example, you can use
futures to write to a log file asynchronously.

This kind of flexibility is pretty cool. Clojure also allows you to
treat "task definition" and "requiring the result" independently with
`delays` and `promises`. Onward!

### Delays

Delays allow you to define a task definition without having to execute
it or require the result immediately. You can create a delay with
`delay`. In this example, nothing is printed:

```clojure
(def jackson-5-delay
  (delay (let [message "Just call my name and I'll be there"]
           (println "First deref:" message)
           message)))
```

You can evaluate the delay and get its result by dereferencing or
using `force`. `force` has the same effect as `deref`; it just
communicates more clearly that you're causing a task to start
as opposed to waiting for a task to finish.

```clojure
(force jackson-5-delay)
; => First deref: Just call my name and I'll be there
; => "Just call my name and I'll be there"
```

Like futures, a delay is only run once and its result is cached.
Subsequent dereferencing will return the Jackson 5 message without
printing anything:

```clojure
@jackson-5-delay
; => "Just call my name and I'll be there"
```

One way you can use a delay is to fire off a statement the first time
one future out of a group of related futures finishes. For example,
pretend your app uploads a set of headshots to a headshot-sharing site
and notifies the owner as soon as the first one is up, as in the
following:

```clojure
(let [notify (delay (email-user "and-my-axe@gmail.com"))]
  (doseq [headshot gimli-headshots]
    (future (upload-document headshot)
            @notify)))
```

In this example, Gimli will be grateful to know when the first
headshot is available so that he can begin tweaking it and sharing it.
He'll also appreciate not being spammed, and you'll appreciate not
facing his reaction to being spammed.

This technique can help protect you from the Mutual Exclusion
Concurrency Goblin. You can view a delay as a resource in the same way
that a printer is a resource. Since the body of a delay is guaranteed
to only ever fire once, you can be sure that only one thread will ever
have "access" to the "resource" at a time. Of course, no thread will
ever be able to access the resource ever again. That might be too
drastic a constraint for most situations, but in cases like the
example above it works perfectly.

### Promises

Promises allow you to express the expectation of a result indepedently
of the task that should produce it and when that task should run. You
create promises with `promise` and deliver a result to them with
`deliver`. You obtain the result by dereferencing:

```clojure
(def my-promise (promise))
(deliver my-promise (+ 1 2))
@my-promise

; => 3
```

You can only deliver a result to a promise once. Just like with
futures and delays, dereferencing a promise will block until a result
is available.

One potential use for promises is to find the first satisfactory
element in a collection of data. Suppose, for example, that you're
gathering ingredients for a spell to summon a parrot that repeats what
people say, but in James Earl Jones's voice. Because James Earl Jones
has the smoothest voice on earth, one of the ingredients is premium
yak butter with a smoothness rating of 97 or greater. Because you
haven't yet summoned a tree that grows money, you have a budget of
$100 for one pound (or 0.45kg if you're a European sorcerer).

Because you are a modern practitioner of the magico-ornithological
arts, you know that yak butter retailers now offer their catalogs
online. Rather than tediously navigate each site, you create a script
to give you the URL of the first yak butter offering that meets your
needs.

In this scenario, the collection of data is the yak butter options
provided by each store. You can model that by defining some yak butter
products, creating a function to mock out an API call, and creating
a function to test whether a product is satisfactory:

```clojure
(def yak-butter-international
  {:store "Yak Butter International"
    :price 90
    :smoothness 90})
(def butter-than-nothing
  {:store "Butter than Nothing"
   :price 150
   :smoothness 83})
;; This is the butter that meets our requirements
(def baby-got-yak
  {:store "Baby Got Yak"
   :price 94
   :smoothness 99})

(defn mock-api-call
  [result]
  (Thread/sleep 1000)
  result)

(defn satisfactory?
  [butter]
  (and (<= (:price butter) 100)
       (>= (:smoothness butter) 97)
       butter))
```

The API call waits 1 second before returning a result to simulate the
time it would take to perform an actual call.

If you check each site serially, it could take more than 3 seconds to
obtain a result, as the following code shows. Note that `time` prints
the time taken to evaluate a form and `comp` composes functions:

```clojure
(time (some (comp satisfactory? mock-api-call)
            [yak-butter-international butter-than-nothing baby-got-yak]))
; => "Elapsed time: 3002.132 msecs"
; => {:store "Baby Got Yak", :smoothness 99, :price 94}
```

You can use a promise and futures to perform each check on a separate
thread. If your computer has multiple cores, you could cut down the
time taken to about 1 second:

```clojure
(time
 (let [butter-promise (promise)]
   (doseq [butter [yak-butter-international butter-than-nothing baby-got-yak]]
     (future (if-let [satisfactory-butter (satisfactory? (mock-api-call butter))]
               (deliver butter-promise satisfactory-butter))))
   (println "And the winner is:" @butter-promise)))
; => "Elapsed time: 1002.652 msecs"
; => And the winner is: {:store Baby Got Yak, :smoothness 99, :price 94}
```

By decoupling the requirement for a result from how the result is
actually computed, you can perform multiple computations in parallel
and save yourself some time.

You can view this as a way to protect yourself from the Reference Cell
Concurrency Goblin. Since promises can only be written to once, you're
can't create the kind of inconsistent state that arises from
nondeterministic reads and writes.

The last thing I should mention is that you can also use promises to
register callbacks, achieving the same functionality that you might be
used to in JavaScript. Here's how to do it:

```clojure
(let [ferengi-wisdom-promise (promise)]
  (future (println "Here's some Ferengi wisdom:" @ferengi-wisdom-promise))
  (Thread/sleep 100)
  (deliver ferengi-wisdom-promise "Whisper your way to success."))
; => Here's some Ferengi wisdom: Whisper your way to success.
```

In this example, you're creating a future that executes immediately.
However, it's blocking because it's waiting for a value to be
delivered to `ferengi-wisdom-promise`. After 100 milliseconds, you
deliver the value and the `println` statement in the future runs.

Futures, delays, and promises are great, simple ways to manage
concurrency in your application. In the next section, we'll look at
one more fun way to keep your concurrent applications under control.

### Simple Queueing

So far you've looked at some simple ways to combine futures, delays,
and promises to make your concurrent programs a little safer. In this
section, you'll use a macro to combine futures and promises in a
slightly more complex manner. You might not necessarily ever use this
code, but it'll show the power of these simple tools a bit more.

Sometimes the best way to handle concurrent tasks is to re-serialize
them. You can do that by placing your tasks onto a queue. In this
example, you'll make API calls to pull random quotes from
[I Heart Quotes](http://www.iheartquotes.com/) and write them to your
own quote library. For this process, you want to allow the API calls
to happen concurrently but you want to serialize the writes so that
none of the quotes get garbled, like the spell example above.

Here's the un-queued quote for retrieving a random quote and writing
it:

```clojure
(defn append-to-file
  [filename s]
  (spit filename s :append true))

(defn format-quote
  [quote]
  (str "=== BEGIN QUOTE ===\n" quote "=== END QUOTE ===\n\n"))

(defn snag-quotes
  [n filename]
  (dotimes [_ n]
    (->> (slurp "http://www.iheartquotes.com/api/v1/random")
         format-quote
         (append-to-file filename)
         (future))))
```

If you call `(snag-quotes 2 "quotes.txt")` you'll end up with a file
that contains something like:

```
=== BEGIN QUOTE ===
Leela: You buy one pound of underwear and you're on their list forever.

[futurama] http://iheartquotes.com/fortune/show/1463
=== END QUOTE ===

=== BEGIN QUOTE ===
It takes about 12 ears of corn to make a tablespoon of corn oil.

[codehappy] http://iheartquotes.com/fortune/show/39667
=== END QUOTE ===
```

Most likely you got lucky like I did and your quotes weren't
interleaved. To ensure that you'll always be this lucky, you can use
this macro:

```clojure
(defmacro queue
  [q concurrent-promise-name & work]
  (let [concurrent (butlast work)
        serialized (last work)]
    `(let [~concurrent-promise-name (promise)]
       (future (deliver ~concurrent-promise-name (do ~@concurrent)))
       (deref ~q)
       ~serialized
       ~concurrent-promise-name)))
```

`queue` works by splitting a task into a concurrent and serialized
portion. It creates a future, which is what allows the concurrent
portion to run concurrently. You can see this with `(future (deliver
~concurrent-promise-name (do ~@concurrent)))`. The next line, `(deref
~q)`, blocks the thread until `q` is done, preventing the serialized
portion from running until the previous job in the queue is done.
Finally, the macro returns a promise which can then be used in another
call to `queue`.

To demonstrate that this works, you're going to pay homage to the
British, since they invented queues. you'll use a queue to ensure that
the customary British greeting, "'Ello, gov'na! Pip pip! Cheerio!" is
delivered in the correct order. This demonstration is going to involve
an abundance of `sleep`ing, so here's a macro to do that more
concisely:

```clojure
(defmacro wait
  "Sleep `timeout` seconds before evaluating body"
  [timeout & body]
  `(do (Thread/sleep ~timeout) ~@body))
```

Now here's an example of the greeting being delivered in the wrong
order:

```clojure
(future (wait 200 (println "'Ello, gov'na!")))
(future (wait 400 (println "Pip pip!")))
(future (wait 100 (println "Cheerio!")))

; => Cheerio!
; => 'Ello, gov'na!
; => Pip pip!
```

This is the wrong greeting completely, though no British person would
be so impolite as to correct you. Here's how you can `queue` it so as
not to embarrass yourself:

```clojure
(time @(-> (future (wait 200 (println "'Ello, gov'na!")))
           (queue line (wait 400 "Pip pip!") (println @line))
           (queue line (wait 100 "Cheerio!") (println @line))))
; => 'Ello, gov'na!
; => Pip pip!
; => Cheerio!
; => "Elapsed time: 401.635 msecs"
```

Blimey! The greeting is delivered in the correct order, and you can
see by the elapsed time that the "work" of sleeping was shunted onto
separate cores.

Now let's use this to read from "I Heart Quotes" concurrently while
writing serially. Here's the final product. I'll walk through it below.

```clojure
(defn append-to-file
  [filename s]
  (spit filename s :append true))

(defn format-quote
  [quote]
  (str "=== BEGIN QUOTE ===\n" quote "=== END QUOTE ===\n\n"))

(defn random-quote
  []
  (format-quote (slurp "http://www.iheartquotes.com/api/v1/random")))

(defmacro snag-quotes-queued
  [n filename]
  (let [quote-gensym (gensym)
        queue-line `(queue ~quote-gensym
                           (random-quote)
                           (append-to-file ~filename @~quote-gensym))]
    `(-> (future)
         ~@(take n (repeat queue-line)))))

```

`append-to-file` and `format-quote` are the same as above, they're
just shown here again so you don't have to flip back  and forth.
`random-quote` simply grabs a quote and formats it.

The interesting bits are in `snag-quotes-queued`. The point of the
macro is to returns a form that looks like:

```clojure
(snag-quotes-queued 4 "quotes.txt")
; => expands to:
(-> (future)
    (queue G__627 (random-quote) (append-to-file "quotes.txt" @G__627))
    (queue G__627 (random-quote) (append-to-file "quotes.txt" @G__627))
    (queue G__627 (random-quote) (append-to-file "quotes.txt" @G__627))
    (queue G__627 (random-quote) (append-to-file "quotes.txt" @G__627)))
```

There are a couple things going on here. You have to "seed" the queue
with a no-op future because `queue` expects a dereferenceable object
as its first argument. Then, you just repeat instances of `(queue
G__627 (random-quote) (append-to-file "quotes.txt" @G__627))`. This
mirrors the way the British example above uses `queue`.

And that's it for futures, delays, and promises! This section has
shown how you can combine them together to make your concurrent
program safer.

In the rest of the chapter, you'll dive deeper into Clojure's
philosophy and discover how the language's design enables even more
power concurrency tools.

## Escaping The Pit of Evil

Literature on concurrency generally agrees that the Three Concurrency
Goblins are all spawned from the same pit of evil: shared access to
mutable state. You can see this in the Reference Cell discussion
above. When two threads make uncoordinated changes to the reference
cell, the result is unpredictable.

Rich Hickey designed Clojure so that it would specifically address the
problems which arise from shared access to mutable state. In fact,
Clojure embodies a very clear conception of state that makes it
inherently safer for concurrency. It's safe all the way down to its
*meta-freakin-physics*.

You'll learn Clojure's underlying metaphysics in this section. To
fully illustrate it, I'll compare it to the metaphysics embodied by
object-oriented languages. I'll also introduce a new type, Clojure's
`atom`. By learning this philosophy, you'll be fully equipped to
handle Clojure's remaining concurrency tools.

When talking about metaphysics things tend to get a little fuzzy, but
hopefully this will all make sense.

As the ever-trusty
[Wikipedia](http://en.wikipedia.org/wiki/Metaphysics) explains,
metaphysics attempts to answer two basic questions in the broadest
possible terms:

* What is there?
* What is it like?

To draw out the differences, let's go over two different ways of
modeling a cuddle zombie. Unlike a regular zombie, a cuddle zombie
does not want to devour your brains. It wants only to spoon you and
maybe smell your neck. That makes its undead, shuffling, decaying
state all the more tragic.

### Object-Oriented Metaphysics

In OO metaphysics, a cuddle zombie is something which actually exists
in the world. I know, I know, I can hear what you're saying: "Uh...
yeah? So?" But believe me, the accuracy of that statement has caused
many a sleepless night for philosophers.

The wrinkle is that the cuddle zombie is always changing. Its body
slowly deteriorates. Its unceasing hunger for cuddles grows fiercer
with time. In OO terms, we would say that the cuddle zombie is an
object with mutable state, and that its state is ever fluctuating. No
matter how much the zombie changes, though, we still identify it as
the same zombie. Here's how you might model and interact with a cuddle
zombie in Ruby:

```ruby
class CuddleZombie
  # attr_accessor is just a shorthand way for creating getters and
  # setters for the listed instance variables
  attr_accessor :cuddle_hunger_level, :percent_deteriorated

  def initialize(cuddle_hunger_level = 1, percent_deteriorated = 0)
    self.cuddle_hunger_level = cuddle_hunger_level
    self.percent_deteriorated = percent_deteriorated
  end
end

fred = CuddleZombie.new(2, 3)
fred.cuddle_hunger_level  # => 2
fred.percent_deteriorated # => 3

fred.cuddle_hunger_level = 3
fred.cuddle_hunger_level # => 3
```

You can see that this is really just a fancy reference cell. It's
subject to the same non-deterministic results in a multi-threaded
environment. For example, if two threads both try to increment Fred's
hunger level with something like `fred.cuddle_hunger_level =
fred.cuddle_hunger_level + 1`, one of the increments could be lost.

You program will be nondeterministic even if you're only performing
*reads* on a separate thread. For example, suppose you're conducting
research on cuddle zombie behavior. You want to log a zombie's hunger
level whenever it reaches 50% deterioration, but you want to do this
on another thread to increase performance. You could do this:

```ruby
if fred.percent_deteriorated >= 50
  Thread.new { database_logger.log(fred.cuddle_hunger_level) }
end
```

...but another thread could change `fred` before the write actually
takes place. That would be unfortunate. You don't want your data to be
inconsistent when recovering from the zombie apocalypse. However,
there's no way to hold on to the state of an object at a specific
moment in time.

Finally, there's no way to express an event that changes both
`cuddle_hunger_level` and `percent_deteriorated` simultaneously. The
result is that it's possible for `fred` to have an inconsistent state:

```ruby
fred.cuddle_hunger_level = fred.cuddle_hunger_level + 1
# At this time, another thread could read fred's attributes and
# "perceive" fred in an inconsistent state
fred.percent_deteriorated = fred.percent_deteriorated + `
```

Still, the fact that the state of the Cuddle Zombie Object and that
Objects in general are never stable doesn't stop us from treating them
as the fundamental building blocks of programs. In fact, this is seen
as an advantage of OOP. It doesn't matter how the state changes, you
can still interact with a stable interface and all will work as it
should.

This conforms to our intuitive sense of the world. The coffee in my
mug is still coffee no matter how much I swirl it around.

Finally, in OOP, objects do things. They act on each other, changing
state all along the way. We call this behavior. Again, this conforms
to our intuitive sense of the world: change is the result of objects
acting upon each other. A Person object pushes on a Door object and
enters a House object.

You can visualize an Object as a box that only holds one thing at a
time. You change its state by replacing its contents. The object's
identity doesn't change no matter how you change its state.

### Clojure Metaphysics

In Clojure metaphysics, we would say that we never encounter the same
cuddle zombie twice. What we see as a discrete *thing* which actually
exists in the world independent of its mutations is in reality a
succession of *atoms* (in the sense of indivisible, unchanging, stable
entities).

Atoms correspond to the idea of *values* in Clojure. It's clear that
numbers are values. How would it make sense for the number 15 to
"mutate" into another number? It wouldn't! All of Clojure's built-in
data structures are likewise values. They're just as unchanging and
stable as numbers.

These atoms are produced by some metaphysical *process*. (This might
sound hand-wavey, but that's because metaphysics is hand-wavey. You'll
see how Clojure implements this idea in a minute.) For example, the
"Next Zombie State" process gets applied to the atom CZ1 to produce
atom CZ2. The process then gets applied to the atom CZ2 to produce
atom CZ3, and so on.

So, the zombie is not a thing in and of itself. It's a concept that we
superimpose on a succession of related atoms. Identity is not
inherent; it is an endowment. We humans choose to make this endowment
because it's a handy way to *refer* to one succession of related atoms
as opposed to some other succession of related atoms.

From this viewpoint, there's no such thing as "mutable state".
Instead, "state" means "the value of an identity at a point in time."

TODO explain a bit more how this is logical

Here's how you might visualize atoms, process, identity, and state:

TODO image goes here

These atoms don't act upon each other and they can't be changed. They
can't _do_ anything. Change is not the result of one object acting on
another. What we call "change" results when a) a process generates a new
atom and b) we choose to associate the identity with the new atom.

How can we do this in Clojure, though? The tools we've introduced so
far don't allow it. To handle change, Clojure uses *reference types*.
Let's look at the simplest of these, the *atom*.

## Atoms

Clojure's atom reference type allows you to endow a succession of
related values with an identity. Here's how you create one:

```clojure
(def fred (atom {:cuddle-hunger-level 0
                 :percent-deteriorated 0}))
```

To get an atom's current state, you dereference it. Unlike futures,
delays, and promises, dereferencing an atom (or any other reference
type) will never block. When you dereference a reference type, it's
like you're saying, "give me this identity's state as it is right
now," so it makes sense that the operation doesn't block. Here's
Fred's current state:

```clojure
@fred
; => {:cuddle-hunger-level 0, :percent-deteriorated 0}
```

In our Ruby example above, I mentioned that data could change when
trying to log it on a separate thread. There's no danger of that when
using atoms to manage state because each state is immutable. Here's
how you could log a zombie's state:

```clojure
(let [zombie-state @fred]
  (if (>= (:percent-deteriorated zombie-state) 50)
    (future (log (:percent-deteriorated zombie-state)))))
```

To update the atom to refer to a new state, you use `swap!`. This
might seem contradictory. You might be shouting at me, "You said atoms
are unchanging!" Indeed, they are! Now, though, we're working with the
atom *reference type*, a construct which refers to atomic values. From
here on out we'll take "atom" to mean "atom reference type" and
"value" to mean "unchanging entity."

`swap!` receives an atom and a function as arguments. `swap!` applies
the function to the atom's current state in order to produce a new
value, and then updates the atom to refer to this new value. The new
value is also returned. Here's how you might increase Fred's cuddle
hunger level:

```clojure
(swap! fred
       (fn [state]
         (merge-with + state {:cuddle-hunger-level 1})))
; => {:cuddle-hunger-level 1, :percent-deteriorated 0}
```

Dereferencing `fred` will return the new state:

```clojure
@fred
; => {:cuddle-hunger-level 1, :percent-deteriorated 0}
```

Unlike Ruby, it's not possible for `fred` to be an inconsistent state.
You can update both "attributes" at the same time:

```clojure
(swap! fred
       (fn [state]
         (merge-with + state {:cuddle-hunger-level 1
                              :percent-deteriorated 1})))
; => {:cuddle-hunger-level 2, :percent-deteriorated 1}
```

In the example above, you passed `swap!` a function which only takes
one argument (`state`). However, you can also pass `swap!` a function
which takes multiple arguments. For example, you could create a
function which takes two arguments, a zombie state and the amount by
which to increase its cuddle hunger level:

```clojure
(defn increase-cuddle-hunger-level
  [zombie-state increase-by]
  (merge-with + zombie-state {:cuddle-hunger-level increase-by}))

(increase-cuddle-hunger-level @fred 10)
; => {:cuddle-hunger-level 12, :percent-deteriorated 1}
```

You then call `swap!` with the additional arguments:

```clojure
(swap! fred increase-cuddle-hunger-level 10)
; => {:cuddle-hunger-level 12, :percent-deteriorated 1}
@fred
; => {:cuddle-hunger-level 12, :percent-deteriorated 1}
```

Or you could express the whole thing using Clojure's built-in functions:

```clojure
(swap! fred update-in [:cuddle-hunger-level] + 10)
; => {:cuddle-hunger-level 22, :percent-deteriorated 1}
```

This is all interesting and fun, but what happens if two separate
threads call `(swap! fred increase-cuddle-hunger-level 1)`? Is it
possible for one of the increments get "lost" the way it did in the
Ruby example?

The answer is no! `swap!` implements "compare-and-set" semantics,
meaning it does the following internally:

1. It reads the current state of the atom
2. It then applies the update function to that state
3. Next, it checks whether the value it read in step 1 is identical to
   the atom's current value
4. If it is, then `swap!` updates the atom to refer to the result of
   step 2
5. If it isn't, then `swap!` *retries*, going through the process
   again with step 1.

The result is that no swaps will ever get lost.

Sometimes you'll want to update an atom without checking its current
value. For example, you may develop a serum to "reset" a cuddle
zombie's hunger level and deterioration. For those cases, there's the
`reset!` function:

```clojure
(reset! fred {:cuddle-hunger-level 0
              :percent-deteriorated 0})
```