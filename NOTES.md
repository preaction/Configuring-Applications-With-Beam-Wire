## Configuration with Subroutines

Configuring a large OO application consists largely of setting object
attributes from some kind of file. To configure a database connection using
DBI, we need a Data Source Name (DSN) (`dbi:mysql:mydatabase`), a user
(`mysql`), and a password (`password`). It's pretty easy to write a
configuration file that looks like this:

    [database]
    dsn=dbi:mysql:mydatabase
    user=mysql
    password=password

But I hate INI files (and TOML confuses me), so let's use YAML instead:

    ---
    database:
        dsn: dbi:mysql:mydatabase
        user: mysql
        password: password

Now we need some code that will make a DBI object (a database handle or DBH)
from our configuration:

    use YAML ();
    use DBI;

    sub get_config {
        return YAML::LoadFile( 'config.yml' );
    }

    sub get_dbh {
        my ( $config ) = @_;
        my $dsn = $config->{database}{dsn};
        my $user = $config->{database}{user};
        my $pw = $config->{database}{password};
        return DBI->connect( $dsn, $user, $pw );
    }

    my $config = get_config();
    my $dbh = get_dbh( $config );

This is a basic way to use a configuration file.

## Configuration with Objects

But we can do better using a more object-oriented approach with Moo attributes:

    package MyApp;
    use Moo;

    has config => (
        is => 'lazy',
        default => sub {
            return YAML::LoadFile( 'config.yml' );
        },
    );

    has dbh => (
        is => 'lazy',
        default => sub {
            my ( $self ) = @_;
            my $config = $self->config;
            my $dsn = $config->{database}{dsn};
            my $user = $config->{database}{user};
            my $pw = $config->{database}{password};
            return DBI->connect( $dsn, $user, $pw );
        },
    );

    package main;
    my $app = MyApp->new;
    my $dbh = $app->dbh;

## ... And Beyond

So that's our database handle. Now we need to add logging to our application.
So we need a new attribute, a new configuration section, and a new subroutine
that builds the log object from the configuration. Now we need to add caching.
So we need another new attribute, a new configuration section, and a new
subroutine that builds the cache object from the configuration. For every new
configurable object, we need a bunch of code to be able to configure it, and
all that code will be disgustingly similar.

Worse, we might need different parts of our application to have access to
different objects: One of our plugins needs a logger and access to the
database, but another needs a logger and the cache object, and a third needs
all three. How can we write a configuration file that will do all these things?

What if there was a generic way to configure any object using a configuration
file? We could then just ask the configuration to give us the object and it
would! Each object would have a name, and then we just ask for the name and we
get the object. Even better, other objects (like our plugins) could ask for
objects by name too, so when we ask for a plugin, we get the plugin with all
the other objects it needs.

This is a standard thing in object-oriented systems, so much it has a name:
Inversion of Control (in that the object you want controls what it needs to be
constructed with) or Dependency Injection (where the dependencies of an object
are injected into it when needed). That's not important. What's important is
that I'm going to show a module that can do it: Beam::Wire.

# Beam::Wire

## Getting Started

With Beam::Wire, we start with a YAML configuration file again, but this time
we define the arguments to our DBI constructor directly

    database:
        $class: DBI
        $method: connect
        $args:
            - dbi:mysql:mydatabase
            - mysql
            - password

Beam::Wire uses `$` to start a special directive. `$class` sets the class of
the object. `$method` sets the constructor method. `$args` defines the
arguments to the constructor. So, the previous configuration translates into
the following Perl:

    DBI->connect(
        'dbi:mysql:mydatabase',
        'mysql',
        'password'
    );

To get this object, we need to create a Beam::Wire object. In an inversion of
control scenario (still not important), this Beam::Wire object is called a
"container", and the objects we get from it are called "services". So we create
our container object and give it the path to the configuration file:

    my $wire = Beam::Wire->new( file => 'config.yml' );

Then we can get our database handle with the `get()` method:

    my $dbh = $wire->get( 'database' );

## Additional Arguments

One of the benefits to using Beam::Wire instead of a custom configuration
system is that with no additional code I can add more arguments to the DBI
constructor, like `RaiseError` to die on errors immediately or `AutoCommit` to
enable/disable auto-commit:

    database:
        $class: DBI
        $method: connect
        $args:
            - dbi:mysql:mydatabase
            - mysql
            - password
            - RaiseError: 1
              AutoCommit: 0

This translates to the following Perl:

    DBI->connect(
        'dbi:mysql:mydatabase',
        'mysql',
        'password',
        {
            RaiseError => 1,
            AutoCommit => 0,
        }
    );

Importantly, I did not have to write any code to support this, I don't have to
come up with some way to describe a hash reference in INI (again, hate INI, and
yes I know TOML exists), and I don't have to know what possible values a user
might want to set.

On the downside, the user has to know how to create a DBI object. There are
some things we can do about that, but first let's configure the rest of our
application.

## Standard Objects

When we configured DBI, we had to specify the class, the constructor method,
and the arguments as an array. This resulted in a bit more verbosity than I
like in my config file. However, for our caching using the CHI module, we can
be more terse.

When an object constructor is named "new", the conventional name for object
constructors, we don't have to specify it. When an object takes a list of
name/value pairs as arguments to it's constructor, we can simply write them in
like this:

    cache:
        $class: CHI
        driver: FastMmap
        root_dir: /tmp/cache

This translates to the following Perl code:

    CHI->new(
        driver => 'FastMmap',
        root_dir => '/tmp/cache',
    );

This is nice because most of Perl's object-oriented libraries (Moo, Moose,
Mojo::Base, etc...) accept a list of name/value pairs to the constructor as the
standard convention. If we build our objects in a standard way, we can have a
prettier configuration file. In fact, there are lots of advantages to having
objects built in a standard way, not the least of which is that we always know
how to call an object's constructor.

## References

To make the most out of a container, we need to put all of our objects inside,
including our main application (which we called `MyApp`, previously). Our main
application depends on the database and cache objects, so we make those into
attributes, like so:

    package MyApp;
    use Moose;

    has dbh => (
        is => 'ro',
        required => 1,
    );

    has cache => (
        is => 'ro',
        required => 1,
    );

Now we can configure our main application by giving it references to our
database and cache objects using a special directive: `$ref`.

    app:
        $class: MyApp
        dbh:
            $ref: database
        cache:
            $ref: cache

Now when we ask Beam::Wire for our `app` object, it will create our `database`
and `cache` objects, give them to the `MyApp->new()` constructor, and give us
the resulting object:

    my $wire = Beam::Wire->new( file => 'config.yml' );
    my $app = $wire->get( 'app' );

References might seem like unnecessary indirection, but now we can configure
multiple instances of our application with the same database and/or cache
settings without repeating ourselves.

This does start raising some red flags in my head: Trying to write a program
with a configuration file is rarely a good idea. So be sure to take care with
these features that you don't create a bigger mess.

## Plugins

Remember when we wanted to be able to configure plugins? Lets add a "plugins"
array to our application object:

    package MyApp;
    use Moo;
    has plugins => (
        is => 'ro',
        default => sub { [] },
    );

I don't know what we're going to do with it, after all, it's not my app
(despite the name), but now we need a plugin. This plugin needs the database,
so let's make sure it gets one:

    package MyApp::Plugin::NeedsDB;
    use Moo;
    has dbh => (
        is => 'ro',
        required => 1,
    );

Rather than create an entirely new, top-level config entry for our plugin, we
can describe it right in the list of plugins:

    app:
        $class: MyApp
        cache:
            $ref: cache
        dbh:
            $ref: database
        plugins:
            -
              $class: MyApp::Plugin::NeedsDB
              dbh:
                $ref: database

The first thing to note is that we can define objects at any place in the
configuration file. This means our configuration file can handle objects that
are arbitrarily complex (if, for some reason, we don't want to simplify them
with a small wrapper class).

Since both `$ref` refer to the same object, both the plugin and the app will
get the exact same database handle. You can verify this with `Scalar::Util`'s
`id` function if you really want.

## Types Constraints

With all this flexibility, it's a good idea to code defensively by declaring
that your attributes must satisfy certain checks. In Perl, these are called
"type constraints", since we don't have static analysis or a compilation step
to ensure type safety. My favorite library for this is currently `Type::Tiny`.
It's pretty easy to use and is compatible with Moo and Moose, the two largest
object libraries in Perl.

Type::Tiny has a library called Types::Standard that contains a lot of useful
type constraints for us to use like `Str` for string and `ArrayRef` for
arrayrefs. Type::Tiny even allows parameters to the types, so we can have an
array ref of objects by saying `ArrayRef[ Object ]`, or an object that inherits
from a specific class using `InstanceOf[ 'MyApp' ]`.

Let's go back and add type constraints to our MyApp attributes: First, the
`database` attribute needs to be an instance of a DBI database handle
(technically a `DBI::db` object).

    package MyApp;
    use Moo;
    use Types::Standard qw( InstanceOf ArrayRef Object );

    has dbh => (
        is => 'ro',
        isa => InstanceOf['DBI::db'],
        required => 1,
    );

The `cache` attribute needs to be an instance of a CHI::Driver object:

    has cache => (
        is => 'ro',
        isa => InstanceOf['CHI::Driver'],
        required => 1,
    );

And the `plugins` attribute needs to be an arrayref of objects:

    has plugins => (
        is => 'ro',
        isa => ArrayRef[ Object ],
        default => sub { [] },
    );

If we don't want to limit ourselves to certain classes for our database and
cache objects, we can create constraints that check that certain methods are
available. This allows our application users to plug in all kinds of new
functionality as long as they satisfy the API we're looking for (which is
similar to a classic object-oriented pattern that Java calls "interfaces", but
properly called "Duck-typing" as in "If it quacks like a duck...").

## Coercions

Rather than having to specify an array of arguments for our DBI object, we can
set up a coercion that will accept a hash reference, exactly like we had in our
first configuration file. 

First, we write the coercion subroutine:

    has dbh => (
        is => 'ro',
        isa => InstanceOf['DBI::db'],
        required => 1,
        coerce => sub {
            my ( $thing ) = @_;
            if ( ref $thing eq 'HASH' ) {
                return DBI->connect( $thing->{dsn}, $thing->{user}, $thing->{password} );
            }
            return $thing;
        },
    );

Then we can take our old configuration:

    database:
        $class: DBI
        $method: connect
        $args:
            - dbi:mysql:mydatabase
            - mysql
            - password

And change it to a simple hashref:

    database:
        dsn: dbi:mysql:mydatabase
        user: mysql
        password: password

The Type::Tiny library includes ways for us to create sharable coercions, but
that's a bit outside the scope of this talk. Suffice it to say that the
attribute could, with some application of the Types::Library module, instead
look like this and still work exactly the same:

    has dbh => (
        is => 'ro',
        isa => MyDatabase,
        required => 1,
        coerce => 1,
    );

Then the `MyDatabase` type could be used by our plugins and other modules to
allow the same coercions.

## Additional Features

Because we have this flexible way of defining real Perl objects, we can also do
some other things like:

### Event Handlers

Using the Beam::Emitter event class, you can create hooks for your application
that users can hook into by defining event handlers in the configuration file
using `$on`:

    app:
        $class: MyApp
        $on:
            - event_name:
                $class: MyEventHandler
                $sub: handle_event

This translates to the following Perl code:

    my $plugin = MyEventHandler->new;
    $app->on( event_name => sub {
        $plugin->handle_event( @_ );
    } );

### Roles

You can compose roles using Role::Tiny right in the configuration file to
extend/override the main object:

    app:
        $class: MyApp
        $with:
            - MyRole

    app2:
        $class: MyApp
        $with:
            - MyOtherRole

This translates to the following Perl code:

    Role::Tiny->apply_roles_to_object( $app, 'MyRole' );

Note that this only changes the given object, not the entire class, so that
each object can have its own set of roles. This is good for arranging sets of
optional features which can be composed ad-hoc by your users.


