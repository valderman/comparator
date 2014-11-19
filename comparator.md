Comparisons in Java
===================

While teaching a course in data structures, relying heavily on Java generics,
I've noticed that many students have problems understanding why and how to use
the [`Comparator`](https://docs.oracle.com/javase/7/docs/api/java/util/Comparator.html) interface and how it relates to [`Comparable`](https://docs.oracle.com/javase/7/docs/api/java/lang/Comparable.html).

Even worse, asking the students to Google for more information on how to use it
does not really work, as a query for "java comparator tutorial" mainly returns
articles that assume more knowledge than the troubled students generally have,
verbose crap that doesn't even talk about actual comparators, and outright
misinformation which confuses the issue even more.

This post attempts to explain how the `Comparator` and `Comparable` interfaces
work and how they differ in an intuitive way.


An example: friend rankings
---------------------------

Imagine that you are throwing a
[tea party](https://www.youtube.com/watch?v=pdEcqSBx4J8) for your friends.
However, your tea house is rather small, so you can't invite all of them.
In this scenario, it might make sense to write a small program to objectively
rank your friends based on some arbitrary criteria, to make sure that you only
invite your `n` bestest friends, where your tea house has space enough for `n`
guests.


First attempt: using `Comparable`
---------------------------------

Let's start off by defining a `Friend` class, to represent each of our friends.
We must be able to compare friends to each other, to determine which out of
two friends is the better one, so we implement the `Comparable` interface.

The `Comparable<T>` interface has a single method: `public int compareTo(T x)`,
which must return a negative number if `this` is less than `x`, `0` if they are
equal, and a positive number if `this` is greater than `x`.

    public class Friend implements Comparable<Friend> {
        public String name;

        // How talkative is this friend, on a scale from 1 to 10?
        public int talkativity;

        // How funny is this friend, on a scale from 1 to 10?
        public int funniness;

        // How kind and understanding is this friend, on a scale from 1 to 10?
        public int kindness;

        public Friend(String name, int talkative, int funny, int kind) {
            this.name = name;
            this.talkativity = talkative;
            this.funniness = funny;
            this.kindness = kind;
        }

        @Override
        public String toString() {return this.name;}

        @Override
        public int compareTo(Friend other) {
            int thisFriendValue = this.talkativity +
                                  this.funniness +
                                  this.kindness;

            int otherFriendValue = other.talkativity +
                                   other.funniness +
                                   other.kindness;

            if(thisFriendValue < otherFriendValue) {
                return -1;
            } else if(thisFriendValue == otherFriendValue) {
                return 0;
            } else {
                return 1;
            }
        }
    }

Note how the `compareTo` method attempts to weigh all of a friend's traits
together in an attempt to create an ordering. This looks a bit clumsy, and
indeed it is, but we don't have much of a choice: `Comparable` has only a
single `compareTo` method and thus only one way of comparing things.

Then, we can define a data structure which takes care of the actual sorting:

    public class SortedFriendList {
        private ArrayList<Friend> friends = new ArrayList<Friend>();

        /**
         * Insert a Friend into the list, ensuring that the best friend in the
         * list is always in the first position.
         */
        public void insert(Friend friend) {
            int insertAt;
            for(int insertAt = 0; insertAt < friends.size(); ++insertAt) {
                if(friend.compareTo(friends.get(insertAt)) > 0) {
                    break;
                }
            }
            friends.add(insBefore, friend);
        }

        /**
         * Returns your nth best Friend.
         */
        public Friend getBestFriend(int n) {
            return friends.get(n);
        }
    }

Finally, we write a `main` method tying these two together:

    public class FriendRanking {
        public static void main(String[] args) {
            SortedFriendList friends = new SortedFriendList();
            friends.insert(new Friend("Jonas", 5, 8, 5));
            friends.insert(new Friend("Ulla",  3, 5, 7));
            friends.insert(new Friend("Rossana", 10, 2, 2));
            System.out.println("Best friend: " + friends.getBestFriend(0));
        }
    }

This program will print `"Best friend: Jonas"`, because our `compareTo` method
weighs together all of a friend's traits and ranks friends who have higher
traits overall higher than those with lower.

Is this really a good way to rank our friends?


A better way: using `Comparator`
--------------------------------

When using `Comparable` as we did in the previous version of our program, we
define a single `compareTo` method, *the One True Way* of comparing two objects.
This makes sense for some things, such as numbers and other objects that have a
canonical total ordering.

In the real world, however, things are rarely so simple. Often, the relative
"goodness" of two objects depends heavily on the situation. Using our friend
ranking example, when inviting people to a tea party we might prefer our
talkative friends and not care too much about whether they are kind,
understanding people or not, while the opposite would be true for when
we determine who to call for comfort after breaking up with a significant
other.

In the functional programming world, this is often solved by using explicit
*comparator functions*, as demonstrated by this Haskell snippet:

    -- Data type representing an ordering of two objects.
    data Ordering = LessThan | Equal | GreaterThan

    -- Data type representing a friend.
    data Friend = Friend {
      talkativity :: Int,
      funniness   :: Int,
      kindness    :: Int
    }

    -- Function that compares two friends based on how talkative they are.
    talkativeComparator :: Friend -> Friend -> Ordering
    talkativeComparator f1 f2 =
      | talkativity f1 > talkativity f2 = GreaterThan
      | talkativity f1 < talkativity f2 = LessThan
      | otherwise                       = Equal

    -- Function that compares two friends based on how funny they are.
    funComparator :: Friend -> Friend -> Ordering
    funComparator f1 f2
      | funniness f1 > funniness f2 = GreaterThan
      | funniness f1 < funniness f2 = LessThan
      | otherwise                   = Equal

    -- Find our most talkative friend.
    mostTalkativeFriend friendlist =
      head (sortBy talkativeComparator friendlist)

    -- Find our funniest friend.
    funniestFriend friendlist =
      head (sortBy funComparator friendlist)

Note how we can sort a list using any ranking criteria we want using the
`sortBy` function together with a comparator function. In this way, the ordering
of two objects is no longer tied to the `Friend` type, but something we can
specify separately as the context requires.

In Java, this concept is expressed using the `Comparator` interface, which is
defined as follows:

    public interface Comparator<T> {
        int compare(T a, T b);
    }

This interface contains a single method, `compare` which takes two arguments
of type `T`, `a` and `b`. If `a` is in some sense "smaller" than `b`,
it must return a negative number, if they are equal it must return `0`,
and if `a` is somehow "larger" than `b` it must return a positive number.
Note that the return value follows the same pattern as for
`Comparable.compareTo`.

Using this interface, we can now define separate ways of ranking our friends:

    public class TeaPartyComparator implements Comparator<Friend> {
        @Override
        public int compare(Friend a, Friend b) {
            if(a.talkativity < b.talkativity) {
                return -1;
            } else if(a.talkativity > b.talkativity) {
                return 1;
            } else {
                return 0;
            }
        }
    }

    public class EmotionalSupportComparator implements Comparator<Friend> {
        @Override
        public int compare(Friend a, Friend b) {
            if(a.kindness < b.kindness) {
                return -1;
            } else if(a.kindness > b.kindness) {
                return 1;
            } else {
                return 0;
            }
        }
    }

We defined two comparators: one which compares friends based on how talkative
they are, making it ideal for deciding who to invite to your tea party, and one
which compares friends based on their kindness, which helps you find a shoulder
to cry on after your cat's been killed in traffic.

Now, let's re-implement our `SortedFriendList` class to make use of
comparators:

    public class SortedFriendList {
        private Comparator<Friend> comp;
        private ArrayList<Friend> friends = new ArrayList<Friend>();

        /**
         * Construct a SortedFriendList using a specific ordering.
         */
        public SortedFriendList(Comparator<Friend> comp) {
            this.comp = comp;
        }

        /**
         * Insert a Friend into the list, ensuring that the best friend in the
         * list is always in the first position.
         */
        public void insert(Friend friend) {
            int insertAt;
            for(int insertAt = 0; insertAt < friends.size(); ++insertAt) {
                if(comp.compare(friend, friends.get(insertAt)) > 0) {
                    break;
                }
            }
            friends.add(insBefore, friend);
        }

        /**
         * Returns your nth best Friend.
         */
        public Friend getBestFriend(int n) {
            return friends.get(n);
        }
    }

There are only two small changes in here: we now keep a private
`Comparator<Friend>` object - that is, a comparator which can compare objects
of the type `Friend` - in our list class, which the user must supply when
creating a new `SortedFriendList`, and we use this comparator's `compare`
method instead of the `Friend` class' `compareTo` method to decide how to order
the list.

The changes to our main program needed to use this new version of our class is
minimal:

    public class FriendRanking {
        public static void main(String[] args) {
            TeaPartyComparator comp = new TeaPartyComparator();
            SortedFriendList friends = new SortedFriendList(comp);
            friends.insert(new Friend("Jonas", 5, 8, 5));
            friends.insert(new Friend("Ulla",  3, 5, 7));
            friends.insert(new Friend("Rossana", 10, 2, 2));
            System.out.println("Best party friend: " + friends.getBestFriend(0));
        }
    }

Now, this program will print "Best party friend: Rossana" instead, because
Rossana is the most talkative of our friends, which is the trait that
`TeaPartyComparator` uses to rank our friends.


Closing remarks
---------------

I hope this example helped illustrate the differences between `Comparable` and
`Comparator`, how to use them, and when to use which of them. To recap:

  * When comparing objects that have *a single objective, canonical ordering*,
    use `Comparable`. Examples: numbers, keys in a dictionary, units of
    measurement.

  * To use `Comparable` with a class, your class should *implement*
    `Comparable<MyClass>` and its `compareTo` method.

  * When comparing objects whose ordering *depends on the context* or is
    subjective, use `Comparator`s. Examples: people, animals, opinions.

  * To use comparators, *create a distinct class* for each possible ordering.
    *Those classes* must implement `Comparator<MyClass>` and its `compare`
    method.
