# The Service Layer Package

The Service Layer package for Joomla provides a basis on which services can be built that enhance
the regular Model-View-Controller (MVC) structure used in components.

The Service Layer provides a clear API for the component, which is something much needed for the
construction of an external REST API.  Ideally, the Service Layer API is the only route through
which outside code accesses the component.  In other words, the Service Layer defines the
component's public boundary and the only operations publicly available are those that are made
available by the Service Layer.

The following problems are addressed:
* multi-channel operation (eg. web, CLI, CoAP, message queue)
* asynchronous operation (eg. via message queues)
* distributed operation (eg. domain events can be published to remote systems)

Whilst the Service Layer presents the public API of a component, the nature of the API is somewhat
different from other APIs that you might be used to working with.
* All inputs and outputs are in the form of message objects.
* Requests are made in the form of immutable message objects.
* Separation of read-only requests from write-only requests.
* Replies to read requests are in the form of data transfer objects (document messages).
* Replies to write requests are in the form of published message objects.

In developing this service layer package an important goal has been to hide the complexity of the
command bus implementation behind a very simple facade that makes it easy to use by extension
developers without the requirement for them to understand how it works.

## Command Query Responsibility Segregation (CQRS)

The service layer inherently supports the idea of Command Query Responsibility Segregation (CQRS)
where "commands" are treated slightly differently from "queries".  A command is some action that
causes the model state to be changed in some way, but produces no output as a result.  On the other
hand, a query causes no changes to model state, but does produce output.  In other words, a
query has no side-effects for which the caller is held responsible.

Carefully designing an extension to separate these concerns can assist with scalability
because it makes it much easier to have separate models for commands and queries that can
be separately optimised.  In the majority of applications there are many more, often orders of
magnitude more, reads than writes.  So the read ("query") models can be optimised to efficiently
deliver data, whereas the write ("command") models can be optimised to ensure data integrity.
This might mean, for example, implementing the write model on a traditional relational database,
whereas the read model uses denormalised views, perhaps on a NoSQL database.

This is not to say that this separation is enforced.  Only that should the developer wish to make
use of CQRS to optimise performance, the scaffolding to do that is readily available.

The separation of commands from queries is embodied in the Service Layer package by having two
separate base classes for the two kinds of message.  These are handled slightly differently by
the default middleware configuration (see below).

## Service Layer as API

The introduction of a service layer directly addresses the problem of defining a component-level API
that can be used across a variety of "channels".  The API is completely defined by the commands
it can handle and the domain events that are raised as a result and by the queries it can handle
and the data returned as a result.

## Installation

* Install Joomla in the usual way.
* Add the GitHub repository
```
composer config repositories.jservice vcs https://github.com/chrisdavenport/service.git
```
* Require the package
```
composer require chrisdavenport/service:dev-master
```

## Using the Service Layer

A simple service layer consists of just three elements.  A command, a command bus and a command handler.
A common example of where you might use it would be a controller that needs to call a model.  Rather
than calling the model directly, the controller instead creates a command object and passes it to the
command bus.  The command bus dispatches the command to the command handler, which then makes calls to
the model.

Equivalently, on the query side the service layer would consist of a query, a query bus and a query
handler.  Actually, there is no problem combining the two so a controller might contain a mix of
command and query calls.

Queries can be called hierarchically and will work as expected since they return immediately with
the data requested.  However, commands are always executed sequentially even if a command is initiated
from within another command.  This prevents problems that would inevitably arise if commands were to be
executed hierarchically.

Commands and queries are simple, lightweight, immutable, value objects.  They can and should contain
simple validation checks in their constructors so that only valid commands (or queries) may be constructed.
Of course, more complex validation rule checks may need to be implemented deeper in the code, but the
first level of validation can take place directly in the command/query classes themselves.  Because
they are such simple objects, commands and queries, together with domain events which are described later,
may be easily serialised and treated as messages in enterprise messaging applications.  Together
with CQRS this begins to address some of the scalability concerns in the Joomla architecture.

Command and query objects are routed, via the command bus to exactly one handler.  The handler can
perform whatever logic is required of it. In the case of a command this will most likely include calls
to the model to update state.  At any time during the command execution one or more domain events may be
raised to indicate that something of significance has occurred.  The command handler must return
these on exit.  All domain events that were raised are then published to all registered listeners
for those events. 

In the case of a query, the query will not change any state that the caller would be held responsible
for.  No domain events may be raised.  The handler must return the data requested and nothing else.

In most cases you can use the default Service class to instantiate a default command bus and simply
use its handler() method to submit commands and queries.  Since the service class is then responsible
for configuring the command bus it can be used as a standard API entry point for external code.

### Using the service layer in the Joomla CMS

A standard configuration of the service layer is available in the Joomla CMS as JService.  This loads
the default command bus configuration except that it uses the standard Joomla event dispatcher for
dispatching domain events.  If you are an extension developer you can override the JService class to
create your own command bus configurations.

Example:
```php
$service = new \JService;
$service->handle(new Mycommand($arg));
```

### Configuration

In order to use the service layer you need to configure a command bus instance using the command bus
builder.  In most cases the default configuration will be suitable, but all aspects of the configuration
may be overridden if required.

The following code will get a default command bus:
```php
use Joomla\Service\CommandBusBuilder;

$commandBus = (new CommandBusBuilder)->getCommandBus();
```

If you intend to raise Domain Events that need publishing, then you can specify the dispatcher to use
as an argument to the builder.  For example, to use the standard Joomla CMS dispatcher, you can use
the following code:
```php
use Joomla\Service\CommandBusBuilder;

$dispatcher = \JEventDispatcher::getInstance();
$commandBus = (new CommandBusBuilder($dispatcher))->getCommandBus();
```

The following text assumes a command bus using the default configuration.  Refer to the section on
advanced command bus configuration if you want to use some non-standard configuration.

## Commands

Commands are simple, immutable, value objects that have only limited behaviour of their own.
Generally, you will want to create your own command objects by extending from the abstract base
"Command" object as that brings in some useful features, mainly relating to immutability.

Because commands are intended to be immutable, setters are not permitted and any (simple)
attempts to change command properties after object construction will result in an exception.

Command object properties may only be set during object construction.  This can be done by
simply setting properties directly.  You must always call the parent constructor *after*
setting properties.  This signals that object construction is completed and no more setting
of properties will be allowed.

By default, the command object has a magic getter that allows access to all previously set
properties, either directly or via a "get\[property\]()" method.  For example, if a property
called "myprop" has been set (in the constructor), then you can get it's value using either
"$command->myprop" or "$command->getMyprop()".  An attempt to get a property that was not
previously set will result in an exception.  To override the default getter, simply create your
own get method, for example, "public function getMyprop()".

As mentioned earlier, simple validation logic can and should be placed in the constructor so
that invalid commands cannot be instantiated.  Of course, validation logic that involves
checking against other objects and/or database access should not be done in the command
constructor, but rather belongs in the command handlers.  In the Joomla CMS at present, most
validation is done in the models, but really it's not a domain model concern and should
really be moved into the service layer.

### A simple example of command usage

In this example, a simple command is created and submitted to the command bus.  This routes it to
a command handler using the simple convention that the substring "Command" is replaced in a
case-sensitive manner by "CommandHandler" in the command class name.
```php
use Joomla\Service\Command;
use Joomla\Service\CommandBusBuilder;
use Joomla\Service\CommandHandler;

// A concrete command.
final class MycomponentCommandDosomething extends Command
{
	public function __construct($arg1, $arg2)
	{
		$this->arg1 = $arg1;
		$this->arg2 = $arg2;

		parent::__construct();
	}
}

// A concrete command handler.
final class MycomponentCommandHandlerDosomething extends CommandHandler
{
	public function handle(MycomponentCommandDosomething $command)
	{
		// Do something here.
	}
}

// Configure the command bus.
$dispatcher = \JEventDispatcher::getInstance();
$commandBus = (new CommandBusBuilder($dispatcher))->getCommandBus();

// Create a command.
$command = new MycomponentCommandDosomething($arg1, $arg2);

// Execute the command.
$commandBus->handle($command);
```

## Queries

Queries are to all intents and purposes exactly like commands.  The difference lies purely
in how they are treated by the command bus.  A query can be dropped onto the command bus at
any time and will execute and return immediately.  This is in contrast to commands, which
will only execute sequentially, one after another.  Queries are expected to return data,
most likely although not required, in the form of a Data Transfer Object (DTO).  On the other
hand, commands cannot return data.  However, commands can raise and publish Domain Events,
which queries can't.

### A simple example of query usage

In this example, a simple query is created and submitted to the command bus.  This routes it
to a query handler using the simple convention that the substring "Query" is replaced in a
case-sensitive manner by "QueryHandler" in the query class name.

Note that the command bus will not publish any domain events raised in this case.
```php
use Joomla\Service\CommandBusBuilder;
use Joomla\Service\Query;
use Joomla\Service\QueryHandler;

// A concrete query.
final class MycomponentQuerySomething extends Query
{
	public function __construct($arg1, $arg2)
	{
		$this->arg1 = $arg1;
		$this->arg2 = $arg2;

		parent::__construct();
	}
}

// A concrete query handler.
final class MycomponentQueryHandlerSomething extends QueryHandler
{
	public function handle(MycomponentQuerySomething $query)
	{
		// Retrieve some data into a data transfer object (DTO) here.

		return $dto;
	}
}

// Configure the command bus.
$commandBus = (new CommandBusBuilder)->getCommandBus();

// Create a query.
$query = new MycomponentQuerySomething($arg1, $arg2);

// Handle the query.
$dto = $commandBus->handle($query);
```

## Domain Events

The service layer provides integrated support for handling domain events.
A Domain Event is something that happened that domain experts care about and they are
introduced here in the form of immutable, value objects that are essentially simple messages
with little or no behaviour of their own.  Unlike commands, which have a one-to-one
relationship with their handlers, domain events are published to all registered
listeners and several mechanisms exist to make it easy to extend functionality by
registering listeners in extension code.

### Naming domain events

Although unenforceable, it is good practice to name events using the past tense.
Ideally, the terms used should be meaningful to domain experts.
For example, the domain event raised after successfully executing a "RegisterCustomer" command,
might be called "CustomerWasRegistered", or perhaps just "CustomerRegistered". 

### Registering a domain event listener

Any number of domain event listeners, including none at all, may be registered for each domain event type.
Different registration methods may be used depending on the context.  The name of the event, where
a name is required, will be the name of the domain event class with an "on" prefix.  For example,
an event class called "CustomerRegistered" will be associated with the event named "onCustomerRegistered".

Note that developers should not make any assumptions about the order in which domain event listeners
are executed.

#### Call by convention

Typically used within a component, this method constructs the name of a domain event listener class
from the name of the domain event class itself and if the class exists, its event method is called.
The domain event publisher will look for the keyword "Event" (not case-sensitive) in the domain event
class name and if found will check to see if a class exists with "Event" replaced by "EventListener"
and register that class as a listener.

For example, in a typical component there might be a domain event called "MycomponentEventSomethinghappened".
The publisher will look for a class called "MycomponentEventListenerSomethinghappened" and call its
"onMycomponentEventSomethinghappened" method, passing the domain event object as the single parameter.

If the component uses the traditional camel-case autoloader, the domain event and domain event listener
classes will be found in the following paths:
```
/com_mycomponent/event/somethinghappened.php
/com_mycomponent/event/listener/somethinghappened.php
```
Here's an example of a domain event listener:
```php
final class MycomponentEventListenerSomethinghappened
{
	/**
	 * Event listener.
	 *
	 * Note that it must be declared as static otherwise you will get
	 * a strict standards error.
	 *
	 * @param   DomainEvent  $event  A domain event object.
	 */
	public static function onMycomponentEventSomethinghappened(DomainEvent $event)
	{
		// Do whatever you want here.
	}
}
```

#### Calling a Joomla plugin

The event publisher will call the event name trigger method in all installed and enabled Joomla plugins
in the "domainevent" plugin group.

For example, a domain event called "Somethinghappened" will cause the "onSomethinghappened" method to be
called for all installed and enabled plugins in the "domainevent" group, passing the domain event object
as the single parameter.
```php
class PlgDomainEventSomethinghappened extends JPlugin
{
	/**
	 * Event listener triggered on a Somethinghappened event.
	 *
	 * @param   DomainEvent  $event  A domain event object.
	 */
	public function onSomethinghappened(DomainEvent $event)
	{
		// Do whatever you want here.
	}
}
```
#### Registering a callback

Any PHP callable may be registered as a domain event listener.  The function or method called must take
a single argument which will be the domain event object.  Any kind of PHP callable may be used.  For full
details see http://php.net/manual/en/language.types.callable.php.

For example, a class called "MyClass" with a method called "MyMethod" may be registered as a listener for
the domain event "SomethingHappened" using the following code, before passing control to the service layer. 
```php
$dispatcher = \JEventDispatcher::getInstance();
$dispatcher->register('onSomethingHappened', array('MyClass', 'MyMethod));
```

#### Registering a closure

An anonymous function, or closure, may be registered as a domain event listener.  The closure must take
a single argument which will be passed the domain event object.

For example, the following code will register the closure shown as a listener for the "SomethingHappened"
domain event:
```php
$dispatcher = \JEventDispatcher::getInstance();
$dispatcher->register('onSomethingHappened', function($event) { echo 'Do something here'; });
```

### Raising a domain event

Command handlers are expected to return a, possibly empty, array of domain events that were raised.  The
command bus will then take care of publishing those events to all registered listeners.  As a convenience,
a couple of methods are available in the CommandHandler class that make it easy to accumulate domain
events and return them.

To raise a domain event, simply instantiate a domain event object and pass it to the command handler's
raiseEvent method.  For example, the following code raises a "Somethinghappened" event, which takes a
couple of arguments, inside a command handler:
```php
$this->raiseEvent(new Somethinghappened($arg1, $arg2));
```
A domain event listener may also raise further domain events by returning them in an array.  In the
following example the listener method raises a domain event and returns it.
```php
final class MycomponentEventListenerSomethinghappened
{
	/**
	 * Event listener.
	 *
	 * Note that it must be declared as static otherwise you will get
	 * a strict standards error.
	 */
	public static function onMycomponentEventSomethinghappened(DomainEvent $event)
	{
		// Do whatever you want here.

		return array(
			new MycomponentEventSomethingElseHappened($arg1, $arg2)
		);
	}
}
```
The order of publication of events should not be relied upon.

### Releasing domain events

Once a command handler has finished executing, control passes back to the command bus.  There needs to be
some mechanism for passing any domain events raised in the command handler back to the command bus where
they can be published.  This is done with the releaseEvents method.  For example, the following code shows
a simple command handler class which raises a couple of domain events then returns then back to the
command bus.
```php
final class DoSomething extends CommandHandler
{
	/**
	 * Command handler.
	 * 
	 * @param   DoSomething  $command  A command.
	 * 
	 * @return  DomainEvent[]
	 *
	 * @throws  RuntimeException
	 */
	public function handle(DoSomething $command)
	{
		// Some logic goes here.
		
		$this->raiseEvent(new SomethingHappened($arg1, $arg2));

		// Some more logic goes here.
		
		$this->raiseEvent(new SomethingElseHappened($arg3));

		// All done.
		return $this->releaseEvents();
	}
}
```
Since many command handlers end by raising an event, the releaseEvents method also takes an
optional event object as an argument.  The code above can be shortened a little as follows:
```php
final class DoSomething extends CommandHandler
{
	/**
	 * Command handler.
	 * 
	 * @param   DoSomething  $command  A command.
	 * 
	 * @return  array of DomainEvent objects.
	 *
	 * @throws  RuntimeException
	 */
	public function handle(DoSomething $command)
	{
		// Some logic goes here.
		
		$this->raiseEvent(new SomethingHappened($arg1, $arg2));

		// Some more logic goes here.

		// All done.
		return $this->releaseEvents(
			new SomethingElseHappened($arg3)
		);
	}
}
```
## Advanced topics

### Command, query and event reserved properties

The following properties are considered reserved in command, query and event
objects and are not available as settable properties in object constructors.
The values associated with these reserved properties are as follows:-

* *constructed*.  Always returns true once the object has been constructed.
* *name*.  The (short) class name of the object.
* *raisedon*.  The time at which the object was instantiated in microseconds
since 1 January 1970.

### Value objects

Commands, queries and events are all considered to be value objects.  A value
object is an immutable object whose identity depends only on the values of its
properties.  In other words, two objects are considered the same if they have
the same value.

Two value objects can be compared to see if they are the same using the *equals*
method.  For example,
```php
$command1 = new SomeCommand($arg1, $arg2);
$command2 = new SomeCommand($arg3, $arg4);

// Are they the same?
echo $command1->equals($command2) ? 'Yes' : 'No';
```
Normally the *raisedon* timestamp is not taken into consideration when comparing
for equality, however for events it is.  Note that the comparison has only microsecond
granularity; that is, two events occurring in the same microsecond will compare as
equal even though they might not be the same actual event.

### Commands and queries are timestamped

Commands and queries extend from the Message class which adds support for a "raisedon"
property which records the time at which the object was instantiated in microseconds
since 1 January 1970.  This can be useful for event logs and the like.

### Firing another command or query from within a command or query handler

Sometimes it is useful to fire off a query from within a command or query handler.
This is made possible by the fact that the command bus is available as a protected
property in all command and query handler classes.  The query executes and returns
immediately.

Sometimes it is tempting to fire a command from within another command and this is
illustrated in the example below.  However, caution should be exercised
as it could potentially break the "golden rule" that only one aggregate should be
modified per request.  Do this only if you are aware of the consequences.

For example,
```php
final class CommandHandlerDoSomething extends CommandHandler
{
	public function handle(CommandDoSomething $command)
	{
		// Do something.

		// Fire off a new command.  This does not execute immediately.
		$this->getCommandBus()->handle(new CommandDoSomethingMore());

		// Fire off a new query.
		$something = $this->getCommandBus()->handle(new QueryForSomething());

		// Do some more stuff.
	}
}
```
Bear in mind that, unlike query execution, execution of the command will only start
after the current command and all its raised domain events have finished executing.

### Firing a command or query from within a domain event listener

Sometimes it is useful to fire off a query from within a domain event listener.
This is made possible by the fact that the second argument to the event handler method is
the command bus.  The query executes and returns immediately.

Sometimes it is tempting to fire another command from within a domain event listener and
this is illustrated in the example below.  However, caution should be exercised
as it could potentially break the "golden rule" that only one aggregate should be
modified per request.  Do this only if you are aware of the consequences.
```php
use Joomla\Service\CommandBus;

final class EventListenerSomethinghappened
{
	/**
	 * Event listener.
	 *
	 * Note that it must be declared as static otherwise you will get
	 * a strict standards error.
	 *
	 * @param   DomainEvent  $event      A domain event.
	 * @param   CommandBus   $container  DI container.
	 * 
	 * @return  array of domain events or null
	 */
	public static function onEventSomethinghappened(DomainEvent $event, CommandBus $commandBus)
	{
		// Do whatever you want here.

		// Execute another command.  This does not execute immediately.
		$commandBus->handle(new DoSomethingElse($event->data));

		// Fire off a new query.
		$something = $commandBus->handle(new QueryForSomething());
	}
}
```
Note that the new command is queued on the command bus and does not begin execution
until after the current command and all its raised domain events have finished
executing.

### Advanced configuration of the command bus

Most aspects of the command bus can be customised should the default configuration not meet
your particular needs.  The command bus implementation is based on League Tactician and
full documentation may be found at http://tactician.thephpleague.com/tweaking-tactician/

#### Command name extractor

The command name extractor determines the name of the command from the command (or query)
object that is passed in.  The default behaviour is to use the name of the class.  The
following options are available, or you can write your own.

* ClassNameExtractor.  Uses the name of the class.  This is the default.
* NamedCommandExtractor.  Expects each command to implement the NamedCommand interface which
provides the command name directly.

In the following example the default command name extractor is overridden with the
NamedCommandExtractor.

```php
use Joomla\Service\CommandBusBuilder;
use League\Tactician\Plugins\NamedCommand\NamedCommandExtractor;

$commandBus = (new CommandBusBuilder)
	->setCommandNameExtractor(new NamedCommandExtractor)
	->getCommandBus()
	;
```

#### Handler locator

The handler locator takes the command name provided by the command name extractor and uses
it to locate the handler that will handle the command.  The default handler locator replaces
"Command" with "CommandHandler" and "Query" with "QueryHandler" in the command name to
determine the name of the command handler class.  It then instantiates that class.  The
following options are available, or you can write your own.

* InMemoryLocator.  This can be configured with an array which maps command names to their
handlers which are assumed to be pre-loaded or auto-loadable.  See Tactician documentation
for more information.
* Callable.  Loads the command handler from a callable.  This is how the default behaviour
is set up.

The following example is actually the default behaviour, but is useful for illustration.

```php
use Joomla\Service\CommandBusBuilder;

$commandBus = (new CommandBusBuilder)
	->setHandlerLocator(
		function($commandName)
		{
			// Break apart the fully-qualified class name.
			// We do this so that the namespace path is not modified.
			$parts = explode('\\', $commandName);
	
			// Get the class name only.
			$className = array_pop($parts);	
	
			// Determine the handler class name from the command class name.
			$handlerName = str_replace('Command', 'CommandHandler', $className);
			$handlerName = str_replace('Query', 'QueryHandler', $handlerName);
	
			// Construct the fully-qualified class name of the handler.
			$serviceName = implode('\\', $parts) . '\\' . $handlerName;
	
			return new $serviceName($this->getCommandBus());
		}
	)
	->getCommandBus()
	;
```

#### Method name inflector

Having located the class that will be called, the name of the method in that class also
needs to be determined.  The default behaviour is to call the handle() method.  The
following options are available, or you can write your own.

* HandleInflector.  Calls the handle() method.  This is the default behaviour.
* HandleClassNameInflector.  Prefixes the command name with "handle" and uses that as the
method name.  This lets you handle multiple commands in the same handler class.
* HandleClassNameWithoutSuffixInflector.  This is the same as HandleClassName but shaves off
the Command suffix from the command name.
* InvokeInflector.  Calls the magic __invoke() method on a callable handler.

In the following example the default method name inflector is overridden with the
HandleClassNameInflector so that a command such as "MyCommand" will be handled by the
"handleMyCommand" method in the "MyCommandHandler" class.

```php
use Joomla\Service\CommandBusBuilder;
use League\Tactician\Handler\MethodNameInflector\HandleClassNameInflector;

$commandBus = (new CommandBusBuilder)
	->setMethodNameInflector(new HandleClassNameInflector)
	->getCommandBus()
	;
```

#### Middleware

Part of the power of using a command bus is that it can be wrapped with custom middleware
to take care of a wide variety of tasks that would otherwise be quite difficult.

The default configuration wraps the basic command bus in two middleware classes.

* CommandLockingMiddleware.  The basic LockingMiddleware plugin ensures that a command
finishes execution before another command is executed.  This allows you to raise a command
during the execution of a command without fear of causing problems.  Since this behaviour
is not desirable for queries, the CommandLockingMiddleware plugin only provides the locking
behaviour for Commands.
* DomainEventMiddleware.  This middleware plugin publishes all pending domain events after
a command has finished executing.  It will not publish domain events following query execution.
You must specify a suitable event dispatcher during command bus configuration for this plugin
to be used.

For example, to use the standard Joomla CMS dispatcher, use the following
code:
```php
use Joomla\Service\CommandBusBuilder;

$dispatcher = \JEventDispatcher::getInstance();
$commandBus = (new CommandBusBuilder($dispatcher))->getCommandBus();
```

The override the default middleware configuration you must pass in an array of middleware
plugins using the setMiddleware method.  The order of these plugins in the array is significant;
the first element in the array will be the outermost plugin to be executed, the last will be
the innermost.  To remove all plugins, pass in an empty array.  The getMiddleware method is
also available so you can get hold of the current middleware configuration, manipulate it
then set it back.

For example, the following code drops all plugins.
```php
use Joomla\Service\CommandBusBuilder;

$commandBus = (new CommandBusBuilder)
	->setMiddleware([])
	->getCommandBus()
	;
```

As a simple example of the power of command bus middleware. the following code demonstrates
how easy it is to add logging for all commands and queries being handled by the command bus.
This is taken almost directly from the example code here: http://tactician.thephpleague.com/middleware/

```php
use Joomla\Service\CommandBusBuilder;

class Logger
{
	public function log($info)
	{
		echo 'LOG: ' . $info . "\n";
	}
}

class LoggingMiddleware implements Middleware
{
	private $logger = null;

	public function __construct(Logger $logger)
	{
		$this->logger = $logger;
	}

	public function execute($message, callable $next)
	{
		$commandClass = get_class($message);

		$this->logger->log('Starting ' . $commandClass);
		$returnValue = $next($message);
		$this->logger->log('Ending ' . $commandClass);

		return $returnValue;
	}
}

$commandBusBuilder = new CommandBusBuilder();

$commandBus = $commandBusBuilder
	->setMiddleware(
		array_merge(
			[new LoggingMiddleware(new Logger)],
			$commandBusBuilder->getMiddleware()
		)
	)
	->getCommandBus()
	;
```
