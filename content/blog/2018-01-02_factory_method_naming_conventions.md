---
author: "Jørgen Ringen"
title: "Design patterns: Factory method naming conventions"
date: 2018-01-02
slug: factory_method_naming_conventions
---

Factory method pattern naming conventions
The [“factory method”](https://en.wikipedia.org/wiki/Factory_method_pattern) is a very useful pattern, but it has som limitations when it comes to documentation.  Constructors “stand out” in API documentation and are easy to spot, but factory methods on the other hand can be hard to find. There are some naming conventions laid out in [effective java](https://www.amazon.com/Effective-Java-3rd-Joshua-Bloch/dp/0134685997) that can reduce this problem and make the factory methods easier to find in an IDE.

---

**`from`**

*Type-conversion*. Takes a single parameter and returns an instance of the same type as the parameter.
`Date d2 = Date.from(d1);`

---

**`of`**

An aggregation method that takes multiple parameters and returns an instance that “incorporate” the parameters. Return type is same as parameters.
`Set<Gender> genders = EnumSet.of(MALE, FEMALE);`

---

**`valueOf`**

More verbose alternative to from and of
`BigInteger.valueOf(Integer.MAX_VALUE);`

---

**`instance`** or **`getInstance`**

Returns an instance that is described by its parameters (if any). Doesn’t have to have the same type as parameters. Return value is typically cached (singleton, etc)
`KeyFactory.getInstance("RSA");`

---

**`create`** or **`newInstance`**

Like instance or getInstance but returns a new instance on each invocation
`Array.newInstance(classObject, length);`

---

**`getType`** or **`type`**

Like getInstance, but used if the factory method in in a different class than the type.
`Collections.list();`

---

**`fooType`**

A more descriptive version than type, where foo adds meaning to the return type
`Collections.singletonList(T value);`