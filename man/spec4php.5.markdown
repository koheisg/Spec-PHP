spec4php(5) -- Spec files
================================

## SYNOPSIS

A typical spec file:

    <?php
    describe "Calculator"
        it "should multiply"
            calculator(1, '*', 10) should equal 10
        end
        it "should divide"
            calculator(4, '/', 2) should equal 2
        end
    end

In this example we have grouped (**describe**) two tests (**it**)
where each one has a single expectation (**should**).


## DESCRIPTION

Spec files are normal PHP source code files with additional syntax to
define `blocks` and `expectations`. By convention they should end in
`.spec.php` or `Spec.php` although this is not a requirement.

The custom syntax for Spec can be made compatible with that of PHP by
using dots "." as a replacement for spaces. This feature is mostly useful
when you don't want your spec files to report errors in an IDE or when
running the files thru a _linting_ process.

In order to transform the custom Spec syntax to valid (and runnable) PHP
source code you'll need to use the `compiler` component of spec4php(3). The
command line tool, spec4php(1), takes care of this process and will
automatically transform the files. By default, Spec will register a custom
PHP stream wrapper with the prefix _spec://_ which will apply the transformation
to any file it references.


## BLOCKS

These are the different blocks supported. All of them are to be terminated with
the `end` keyword or enclosed in curly braces.

  * `describe` "<var>description</var>":
    This block allows to define a group of tests. These blocks can be
    nested inside other `describe` blocks. They can be think of as the
    equivalent of a PHPUnit suite.

  * `it` "<var>description</var>":
    This is basically a test case. These blocks MUST appear inside `describe`
    blocks. Inside `it` blocks you should include your test logic an apply
    expectations over it. This is the equivalent to a PHPUnit test case method.

  * `before`:
    This block MUST appear inside a `describe` one. It's used to setup the
    _world_ for the tests contained in the parent `describe` block.

  * `after`:
    This block MUST appear inside a `describe` one. It's used to restore or
    clean up the _world_ after all the tests contained in the parent `describe`
    block have been run.

  * `before_each`:
    Almost the same as the `before` block but this one is run just before every
    test (`it` block).

  * `after_each`:
    Almost the same as the `after` block but this one is run just after every
    test (`it` block).


Note that `before_each` and `after_each` are inherited in nested `describe`
groups.


## TEST BLOCKS ##

Test blocks are defined with the `it` keyword. These blocks should test
your code by performing expectations over its functionality. They are equivalent
to PHPUnit's TestCase class methods. In fact, they are wrapped at runtime
in a PHPUnit TestCase class.

You can even access the test case class methods by using the `$this` variable,
as you would in a class method. Spec will automatically translate occurrences
of `$this` inside code blocks and convert them to a call to the underlying
test case class instance. This allows to mix the usage of Spec's native
expectations and original PHPUnit assertions.

    it "should use asserts"
        $this->assertEquals(1, 1);
    end

This feature comes really handy when you have an extended PHPUnit test case
class, like for example with the Zend_Test classes, without having to port
those custom assertions to Spec matchers. Besides, it will ease the path
when converting PHPUnit test cases to Spec, since most of the code used
in your current tests will run unmodified when placed in a spec file.

An additional feature of `it` blocks is their ability to get arguments
for the test from their own description string. Any _value_ enclosed in
quotes (single or double) or between angle brackets (<>) will be made
available inside the code block with variables named `$argX`, where X
is an integer count starting at 1.

    it "should multiply '100' by '10' and return <var>1000</var>" {
        $arg1 * $arg2 should equal $arg3;
    }

This feature is specially suited to create _dynamic_ tests, something that
is a bit more difficult with other testing frameworks. Since test code blocks
are converted to simple anonymous functions, it's possible to wrap them in
loops to feed a set of data to test. The following example will create 5
individual tests feeding different data to each one of them.

    $results = array(0, 100, 200, 300, 400);
    foreach ($results as $i=>$result) {
        it "should multiply '$i' by '100' and return '{$result[$i]}'"
            $arg1 * $arg2 should equal $arg3;
        end
    }


## WORLD ##

All the code blocks (`it`, `before`, `after`, `before_each` and `after_each`)
get automatic access to an object thru a variable named `$W` (uppercase W)
representing the _world_. Hooks like `before` or `before_each` can be used
to configure this _world_ by initializing variables, resources, database
connections, mocks...

Spec will create and restore _snapshots_ of this object so that every test
(`it` blocks) in a `describe` group receives the exact same values configured
in their _world_ object. There are some things that can't be automatically
restored by Spec though, like database connections or file resources, so try
to configure them in `before` hooks instead of `before_each` ones.

Note that since the _world_ is restored after each test is ran, it's not
possible to pass values using it from one test to another. You should always
use the hooks to configure the _world_, otherwise you would run into problems
when tests are executed in a different order or skipped for some reason.

    describe "World"

        before
           $W->foo = 'bar';
        end

        before_each
            $W->foo .= 'baz';
        end

        it "should get an initialized world" {
            $W->foo should equal "barbaz";
        }
    end


## EXPECTATIONS ##

Expectations are defined in Spec by using a subject-predicate form that mimics
english natural language. Basically they take the form "`subject` _should_ `predicate`"
where `subject` is a PHP expression and `predicate` defines matchers and expected
values.

Any PHP expressions can be used before _should_, however some are not completely
supported, for example, it's not possible to use anonymous functions as the
expectation `subject`. To improve readability and ensure the parser works as
expected is useful to wrap them in parenthesis.

Matchers in the `predicate` part can have an expected value, any simple PHP
expression following the matcher phrase idents will be used as an argument to
the matcher function. If you need to use function calls or other more complex
expressions you can wrap them in parenthesis, otherwise the parser might not
be able to parse it correctly.

Expectations do not need to be ended with a semicolon character (';') when the
next word is the `end` keyword or there is an empty line below it.

In some cases it makes sense to use comparison symbols instead of writing it
as text. See the following table for the mapping between the comparison symbols
and their matchers.

       Symbol     |     Matcher
    ------------------------------
        ===       |    same
        !==       |    not same
        ==        |    equal
        !=        |    not equal
        >         |    greater
        <var>         |    less
        </var>=        |    at least
        <=        |    at most

Additionally, any matcher can be negated by using the word `not` in it.

See the following examples of expectations:

    $result should be integer;
    (1+1) should not equal 1;
    trim("  foo ") should be exactly "foo";
    count(array(1,2,3)) should >= 2;
    $result should equal (1/2 + 5);
    1 should not equal 2;
    1 should != 2;
    "foo" should equal (trim("  foo  "))
    true should be ((bool)$var)


## COORDINATION ##

Complex expectations can be _coordinated_ by using operators `and`, `or` and
`but`. It's important to understand the operator precedence rules before
using them, although they try to follow common conventions for the english
language there might be cases where they don't quite do what they look like.

All operators are left-associative and take two operands, thus the precedence
rules are very simple:

      operator  |  precedence index
    ---------------------------------
        and     |        3
        or      |        2
        but     |        1
        ,and    |        1

Please note that it's not possible to override the standard precedence rules
by using parentheses. Expectations should be kept simple, when in doubt break
up complex expectations into simpler ones.

Please review the following examples to see how these precedence rules
apply.

    should be integer or string and equal "1"
    (integer) OR (string AND equal "1")

    -- Note that a comma followed by an operand behaves like an "or"
    should be integer, float or string
    (integer) OR (float) OR (string)
    should be integer, string and equal to 10 or float
    (integer) OR (string AND equal 10) OR (float)

    -- Note that a comma followed by "and" behaves like a "but"
    should be integer or string but less than 10
    should be integer or string, and less than 10
    (integer OR string) AND (less than 10)

    should be integer or string and equal 0 or float
    (integer) OR (string AND equal 0) OR (float)

    should be integer or string and equal "1" but not be a float
    ( (integer) OR (string AND equal "1") ) AND (not be float)

    -- Note that if no matchers are given the last one is used
    should be equal to 10, 20 or 30
    (equal 10) OR (equal 20) OR (equal 30)


## ANNOTATIONS ##

Annotations can be defined in two ways, using the standard javadoc like
comment with `@tag` entries or a more lightweight alternative using
a hash line comment followed by a word: `# tag`.

Most annotations are inherited by child `describe` groups and `it`
blocks. In the case where there is a collision the deepest one in
the hierarchy wins.

Spec understands the following annotation tags:

  * `class` <var>class_name</var>:
    Tells Spec to create a test case inheriting from the given class.
    This is very useful to allow the use of Spec with custom TestCase
    classes you might already have or for enabling the use of Zend_Test
    or PHPUnit's Selenium test case implementation.

  * `throws` [_code_] <var>class</var> [<var>message</var>]:
    This annotation instructs Spec to perform an additional assertion
    when runnning the test, ensuring that it should throw an exception
    matching the given code or the given exception class.

  * `todo`, `incomplete`:
    Flags a test case as incomplete. Spec will report these test cases
    in a different way to standard ones, so it's easy to know when a
    test is passing but doesn't yet tests all the functionality it should.

  * `skip`:
    A test case with this tag will make Spec skip its execution but log
    in the report that it was skipped. It's a great way to disable some
    test cases known to fail for any reason.

Additionally, most PHPUnit annotations should work when using spec files
too, see [PHPUnit documentation](http://www.phpunit.de/manual/current/en/appendixes.annotations.html)


## GOTCHAS

Spec will load the spec files via a custom stream wrapper which provokes
`__DIR__` and `__FILE__` magic constants to include the stream prefix in
them. Often times it's needed to load files relative to the spec file
location, in these cases we would usually use the `__DIR__` constant. Spec
takes this into account and will automatically convert this constant to
calls to `Spec::dir(__DIR__)` which returns a normalized version of the
value. For `__FILE__` however there is no special threatment, so if you
use code like `dirname(__FILE__)` please update it to use the `__DIR__`
one.

    // /path/to/fixtures/class.php
    include __DIR__ . '/fixtures/class.php';
    // /path/to/fixtures/data.txt
    $data = file_get_contents(__DIR__ . '/fixtures/data.txt');
    // spec://fixtures/data.txt
    $data = file_get_contents(dirname(__FILE__) . '/fixtures/data.txt');


Note that including spec files from another spec file is not officially
supported yet. It might work in some cases but it's desirable to layout
your tests using individual files to avoid conflicts and erroneous behavior.


## CUSTOM TEST CLASSES

It's possible to use custom test case classes that extend the
`PHPUnit_Framework_TestCase` one. They can be implemented by you or come
from a framework, like the ones from Zend_Test.

Spec is able to _patch_ any given class to add support for its features, so
it's completely possible to use those classes without having to modify them in
any way.

The way to tell Spec what class it should use is by defining an annotation
for a `describe` or `it` block, like in the following example:

    # class Zend_Test_PHPUnit_ControllerTestCase
    describe "Calculator"
      it "should multiply" {
        (1*3) should equal 3;
      }

      // @class PHPUnit_Framework_TestCase
      it "should divide"
        (3/1) should equal 3;
      end
    end

Note that this annotation is inherited by child blocks, so there is no need
to specify it for each test.



## EXAMPLES ##

See Spec's own tests in the GitHub repository to see examples.


## COPYRIGHT ##

Spec for PHP is Copyright (C) 2011 Ivan -DrSlump- Montes <http://pollinimini.net>


## SEE ALSO

spec4php(1), spec4php(3),
<http://github.com/drslump/spec-php>


[SYNOPSIS]: #SYNOPSIS "SYNOPSIS"
[DESCRIPTION]: #DESCRIPTION "DESCRIPTION"
[BLOCKS]: #BLOCKS "BLOCKS"
[TEST BLOCKS]: #TEST-BLOCKS "TEST BLOCKS"
[WORLD]: #WORLD "WORLD"
[EXPECTATIONS]: #EXPECTATIONS "EXPECTATIONS"
[COORDINATION]: #COORDINATION "COORDINATION"
[ANNOTATIONS]: #ANNOTATIONS "ANNOTATIONS"
[GOTCHAS]: #GOTCHAS "GOTCHAS"
[CUSTOM TEST CLASSES]: #CUSTOM-TEST-CLASSES "CUSTOM TEST CLASSES"
[EXAMPLES]: #EXAMPLES "EXAMPLES"
[COPYRIGHT]: #COPYRIGHT "COPYRIGHT"
[SEE ALSO]: #SEE-ALSO "SEE ALSO"


[spec4php(1)]: spec4php.1.html
[spec4php(3)]: spec4php.3.html
[spec4php(5)]: spec4php.5.html
[ronn]: http://rtomayko.github.com/ronn
[phpunit]: http://phpunit.de
