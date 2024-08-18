Satisfaction Guaranteed

Converting asyncronous operations to synchronous ones safely

The Use Case

We receive new messages via HTTP calls that we want to process as a batch before submitting to a downstream service.

Step One

The simplest way to convert asynchronous to synchronous is by using a queue. We can add new events we aren't ready to process yet to the back of the queue. CE and FS2 provide some constructs which make doing this easier.

(Snippet with channel, processing, and returning the offer func)

Our endpoints call `process: IO[Unit]` which moves messages to the queue and allows the control flow to continue. The stream that results from the queue quietly processes our messages.

This code has a downside. CE IO provides a feature called "cancellation" which can stop the program running for a number of reasons. This may be something as simple as a timeout `IO.sleep(2.seconds).println("I will never print").timeout(1.second)`, a process signal, or cancellation is invoked manually. `IO.cancel?`. We might not willingly cancel our message code but in cloud environments it's common for services to be halted for any number of reasons (deployment, down-scaling, application failure). In this case a cancelation means our code will drop all of the in-memory messages in our queue, which is unlikely to be what we want!

IO and it's cousin Resource both have built-in methods which can guarantee certain actions will run regardless of cancellation. FS2's channel has a special guarantee that, if closed, it will empty the current queue. The hard part is instructing our current code to ensure this channel is closed in response to a cancellation.

// Add guarantee, add uncancelable
// Do we need the IO.uncancelable?

Error Handling

Guaranteeing also makes sure the same code runs even in the event of error. Inversion of control, which does what? This has a downside;

 the sync op could be bad (slow, throws errors)
 this could take down the whole stream processing thread if we don't protect it
 whose responsibility is it? how long should the stream try before giving up?
 can we ever really guarantee delivery?
 given enough time some deployers will forcefully end a service's life
