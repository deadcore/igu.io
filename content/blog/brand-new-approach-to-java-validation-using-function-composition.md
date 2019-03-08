+++
author = "Jack Liddiard"
categories = ["Java", "Functional Programing"]
tags = ["tutorial"]
date = "2018-01-29"
description = "The task was simple but no one provided a testable, scalable and ultimately readable approach."
featured = "pic03.jpg"
featuredalt = "Pic 3"
featuredpath = "date"
linktitle = ""
title = "Brand New Approach to Java Validation Using Function Composition"
type = "post"
+++

I've got to say, I thought the **how to write Java validation** was done to death until I had to write one myself. The task was simple but no one provided a testable, scalable and ultimately readable approach.

Without sounding brash they were all based around the died to death and broken mechanism of:

* Create an `interface` called `ValidationRule` with a single method called `validate`.
* Iterate over a collection of `ValidationRule` and pass your entity in
* Collect the results to a list

I whipped up this quick example to highlight how this works

Fancy skipping the reading and taking me to the [code](https://github.com/deadcore/composite-java-validation-example)

## Proposed Imperative Solution

Our entity we are going to validate is a simple `User`
```java
// User.java
public class User {

    private final String firstname;
    private final String lastname;
    private final int age;
    private final String email;
    private final String username;
    private final boolean hasParentsConsent;

    public User(final String firstname, final String lastname, final int age, final String email, final String username, final boolean hasParentsConsent) {
        this.firstname = firstname;
        this.lastname = lastname;
        this.age = age;
        this.email = email;
        this.username = username;
        this.hasParentsConsent = hasParentsConsent;
    }

    public String getFirstname() {
        return firstname;
    }

    public String getLastname() {
        return lastname;
    }

    public int getAge() {
        return age;
    }

    public String getEmail() {
        return email;
    }

    public String getUsername() {
        return username;
    }

    public boolean isHasParentsConsent() {
        return hasParentsConsent;
    }
}
```

Some basic rules we want to define
```java
// Rules.java
public enum Rules {
    UNDERAGE,
    USERNAME_TAKEN,
    EMAIL_INUSE
}
```

Our validation rules
```java
// ValidationRule.java
public interface ValidationRule {
    Optional<Rules> validate(User user);
}

// UsernameInUseValidator.java
public class UsernameInUseValidator implements ValidationRule {
    @Override
    public Optional<Rules> validate(final User user) {
        if ("Jack".equalsIgnoreCase(user.getUsername())) {
            return Optional.of(Rules.USERNAME_TAKEN);
        } else if ("Deadcore".equalsIgnoreCase(user.getUsername())) {
            return Optional.of(Rules.USERNAME_TAKEN);
        }
        return Optional.empty();
    }
}

// EmailInUseValidator.java
public class EmailInUseValidator implements ValidationRule {
    @Override
    public Optional<Rules> validate(final User user) {
        if ("bill.gates@microsoft.com".equalsIgnoreCase(user.getEmail())) {
            return Optional.of(Rules.EMAIL_INUSE);
        }
        return Optional.empty();
    }
}
```

Finally our application which validates
```java
// Application.java
public class Application {

    public static void main(String... args) {

        final User user = new User(
                "Bill",
                "Gates",
                13,
                "bill.gates@microsoft.com",
                "deadcore",
                false
        );

        final List<ValidationRule> rules = Arrays.asList(
                new EmailInUseValidator(),
                new UsernameInUseValidator()
        );

        final Set<Rules> brokenRules = rules.stream().reduce(new HashSet<>(), (x, y) -> {
            final HashSet<Rules> mutation = new HashSet<>(x);
            final Optional<Rules> maybeBrokenRule = y.validate(user);
            maybeBrokenRule.ifPresent(mutation::add);
            return mutation;
        }, (x, y) -> {
            final HashSet<Rules> mutation = new HashSet<>(x);
            mutation.addAll(y);
            return mutation;
        });

        System.out.println(brokenRules); // Prints [EMAIL_INUSE, USERNAME_TAKEN]

    }

}
```

### Why this sucks
Personally I think the above code is ugly, cumbersome, inflexible, hard to read... The list goes on


## Function compersition
So lets now take a look at how we are going to

First we a re going to define out triggers, attributes which will ultimately make up a validation rule.

```java
// Trigger.java
public interface Trigger extends Predicate<User> {

    Trigger isBillGatesEmail = user -> "bill.gates@microsoft.com".equalsIgnoreCase(user.getEmail());

    Trigger isUsernameJack = user -> "Jack".equalsIgnoreCase(user.getUsername());
    Trigger isUsernameDeadcore = user -> "Deadcore".equalsIgnoreCase(user.getUsername());

    Trigger isUsernameInUse = isUsernameDeadcore.or(isUsernameJack);

    default Trigger or(final Trigger other) {
        return callAttributes -> this.test(callAttributes) || other.test(callAttributes);
    }
}
```
Now lets take a look at composing our validation rules
```java
// UserValidation.java
import java.util.Collections;
import java.util.HashSet;
import java.util.Set;
import java.util.function.Function;
import java.util.function.Predicate;

import static java.util.Collections.emptySet;

public interface UserValidation extends Function<User, Set<Rules>> {
    UserValidation empty = $ -> emptySet();

    UserValidation usernameTaken = rule(Trigger.isUsernameInUse, Rules.USERNAME_TAKEN);

    UserValidation emailTaken = rule(Trigger.isBillGatesEmail, Rules.EMAIL_INUSE);

    static UserValidation rule(final Predicate<User> predicate, final Rules rule) {
        return user -> predicate.test(user) ? Collections.singleton(rule) : emptySet();
    }

    default UserValidation and(final UserValidation other) {
        return user -> {
            final Set<Rules> left = this.apply(user);
            final Set<Rules> right = other.apply(user);

            final Set<Rules> merged = new HashSet<>(left);
            merged.addAll(right);

            return merged;
        };
    }
}
```
Now finally our main application
```java
public class Application {

    public static void main(String... args) {

        final User user = new User(
                "Bill",
                "Gates",
                13,
                "bill.gates@microsoft.com",
                "deadcore",
                false
        );

        final UserValidation validation = UserValidation.empty.
                and(UserValidation.emailTaken).
                and(UserValidation.usernameTaken);

        System.out.println(validation.apply(user)); // [EMAIL_INUSE, USERNAME_TAKEN]
    }

}
```

### Summary & Enhancements

For me the functional mechanism is a lot simpler to read, each function is well isolated only focusing on what it needs to. I don't have to worry about anything outside of the scope I am focusing on, as each function is doing exactly as it should.

We're not doing complex reduce statements where we are having to worry about combiners. Simply put we have written a very light weight and yet very enhanceable DSL for our purpose. We can leverage that but doing some of the following:

#### Or
This becomes apparent if we want to add an `or` operator into `UserValidation`
```java
default UserValidation or(final UserValidation other) {
    return user -> {
        final Set<Rules> left = this.apply(user);

        if (left.isEmpty()) {
            return other.apply(user);
        }

        return left;
    };
}

final UserValidation validation = UserValidation.empty.
        and(UserValidation.emailTaken).
        or(UserValidation.usernameTaken);
```
If executed the above code will only print the `EMAIL_INUSE` rule.

#### Not
Notting a trigger becomes trivial:
```java
Trigger hasParentsConsent = User::isHasParentsConsent;
Trigger noConsent = hasParentsConsent.negate();

default Trigger negate() {
    return user -> !this.test(user);
}
```

#### Testing

Testing becomes trivial, since each method is well defined and isolated, our test cases becomes tiny. Observe

```java
@DisplayName("Trigger")
class TriggerTest
{

   @Nested
   @DisplayName("isBillGatesEmail")
   class IsBillGatesEmail
   {
      private final Trigger trigger = Trigger.isBillGatesEmail;
      private final User user = new User(); // Fill in the blanks

      @Test
      @DisplayName("Should return true if bill gates email")
      void shouldReturnTrueIfBillGates()
      {
         assertTrue(trigger.test(user));
      }

      @Test
      @DisplayName("Should return false if not bill gates email")
      void shouldReturnFalseIfNotBillGates()
      {
         assertFalse(trigger.test(user));
      }
   }

   @Nested
   @DisplayName("and")
   class And
   {
      final Trigger failing = x -> true;
      final Trigger passing = x -> false;

      @Test
      @DisplayName("Should return false if only the left hand side fails")
      void shouldReturnFalseIfLeftSideFails()
      {
         final Trigger trigger = failing.and(passing);

         assertFalse(trigger.test(landline));
      }

      @Test
      @DisplayName("Should return false if only the right hand side fails")
      void shouldReturnFalseIfRightSideFails()
      {
         final Trigger trigger = passing.and(failing);

         assertFalse(trigger.test(landline));
      }

      @Test
      @DisplayName("Should return false if only neither side fails")
      void shouldReturnFalseIfNeitherSideFails()
      {
         final Trigger trigger = passing.and(passing);

         assertFalse(trigger.test(landline));
      }

      @Test
      @DisplayName("Should return true if both sides fail")
      void shouldReturnAnAValidValidationResultNeitherFail()
      {
         final Trigger trigger = failing.and(failing);

         assertTrue(trigger.test(landline));
      }
   }
}
```

## Summary

Throughout the whole article I tried my best not to preach about functional programming. Like most things it has a time and a place, however that being said after seeing how easy, expandable and well defined a basic functional validation framework can be I highly recommend you try it and never look back

Github: https://github.com/deadcore/composite-java-validation-example
