:mod:`zope.security.decorator`
===============================

.. testsetup::

   from zope.component.testing import setUp
   setUp()

.. autoclass:: zope.security.decorator.DecoratedSecurityCheckerDescriptor
   :members:
   :member-order: bysource

To illustrate, we'll create a class that will be proxied:

.. doctest::

   >>> class Foo(object):
   ...     a = 'a'

and a class to proxy it that uses a decorated security checker:

.. doctest::

   >>> from zope.security.decorator import DecoratedSecurityCheckerDescriptor
   >>> from zope.proxy import ProxyBase
   >>> class Wrapper(ProxyBase):
   ...     b = 'b'
   ...     __Security_checker__ = DecoratedSecurityCheckerDescriptor()

Next we'll create and register a checker for `Foo`:

.. doctest::

   >>> from zope.security.checker import NamesChecker, defineChecker
   >>> fooChecker = NamesChecker(['a'])
   >>> defineChecker(Foo, fooChecker)

along with a checker for `Wrapper`:

.. doctest::

   >>> wrapperChecker = NamesChecker(['b'])
   >>> defineChecker(Wrapper, wrapperChecker)

Using `selectChecker()`, we can confirm that a `Foo` object uses
 `fooChecker`:

.. doctest::

   >>> from zope.security.checker import selectChecker
   >>> foo = Foo()
   >>> selectChecker(foo) is fooChecker
   True
   >>> fooChecker.check(foo, 'a')
   >>> fooChecker.check(foo, 'b')  # doctest: +ELLIPSIS
   Traceback (most recent call last):
   ForbiddenAttribute: ('b', <zope.security.decorator.Foo object ...>)

and that a `Wrapper` object uses `wrappeChecker`:

.. doctest::

   >>> wrapper = Wrapper(foo)
   >>> selectChecker(wrapper) is wrapperChecker
   True
   >>> wrapperChecker.check(wrapper, 'b')
   >>> wrapperChecker.check(wrapper, 'a')  # doctest: +ELLIPSIS
   Traceback (most recent call last):
   ForbiddenAttribute: ('a', <zope.security.decorator.Foo object ...>)

(Note that the object description says `Foo` because the object is a
proxy and generally looks and acts like the object it's proxying.)

When we access wrapper's ``__Security_checker__`` attribute, we invoke
the decorated security checker descriptor. The decorator's job is to make
sure checkers from both objects are used when available. In this case,
because both objects have checkers, we get a combined checker:

.. doctest::

   >>> from zope.security.checker import CombinedChecker
   >>> checker = wrapper.__Security_checker__
   >>> type(checker)
   <class 'zope.security.checker.CombinedChecker'>
   >>> checker.check(wrapper, 'a')
   >>> checker.check(wrapper, 'b')

The decorator checker will work even with security proxied objects. To
illustrate, we'll proxify `foo`:

.. doctest::

   >>> from zope.security.proxy import ProxyFactory
   >>> secure_foo = ProxyFactory(foo)
   >>> secure_foo.a
   'a'
   >>> secure_foo.b  # doctest: +ELLIPSIS
   Traceback (most recent call last):
   ForbiddenAttribute: ('b', <zope.security.decorator.Foo object ...>)

when we wrap the secured `foo`:

.. doctest::

   >>> wrapper = Wrapper(secure_foo)

we still get a combined checker:

.. doctest::

   >>> checker = wrapper.__Security_checker__
   >>> type(checker)
   <class 'zope.security.checker.CombinedChecker'>
   >>> checker.check(wrapper, 'a')
   >>> checker.check(wrapper, 'b')

The decorator checker has three other scenarios:

   - the wrapper has a checker but the proxied object doesn't
   - the proxied object has a checker but the wrapper doesn't
   - neither the wrapper nor the proxied object have checkers

When the wrapper has a checker but the proxied object doesn't:

.. doctest::

   >>> from zope.security.checker import NoProxy, _checkers
   >>> del _checkers[Foo]
   >>> defineChecker(Foo, NoProxy)
   >>> selectChecker(foo) is None
   True
   >>> selectChecker(wrapper) is wrapperChecker
   True

the decorator uses only the wrapper checker:

.. doctest::

   >>> wrapper = Wrapper(foo)
   >>> wrapper.__Security_checker__ is wrapperChecker
   True

When the proxied object has a checker but the wrapper doesn't:

.. doctest::

   >>> del _checkers[Wrapper]
   >>> defineChecker(Wrapper, NoProxy)
   >>> selectChecker(wrapper) is None
   True
   >>> del _checkers[Foo]
   >>> defineChecker(Foo, fooChecker)
   >>> selectChecker(foo) is fooChecker
   True

the decorator uses only the proxied object checker:

.. doctest::

   >>> wrapper.__Security_checker__ is fooChecker
   True

Finally, if neither the wrapper not the proxied have checkers:

.. doctest::

   >>> del _checkers[Foo]
   >>> defineChecker(Foo, NoProxy)
   >>> selectChecker(foo) is None
   True
   >>> selectChecker(wrapper) is None
   True

the decorator doesn't have a checker:

.. doctest::

   >>> wrapper.__Security_checker__
   Traceback (most recent call last):
     ...
   AttributeError: 'Foo' has no attribute '__Security_checker__'

 __Security_checker__ cannot be None, otherwise Checker.proxy blows
 up:

   >>> checker.proxy(wrapper) is wrapper
   True


.. autoclass:: zope.security.decorator.SecurityCheckerDecoratorBase
   :members:
   :member-order: bysource


.. autoclass:: zope.security.decorator.DecoratorBase
   :members:
   :member-order: bysource




.. testcleanup::

   from zope.component.testing import tearDown
   tearDown()