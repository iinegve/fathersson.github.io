---
title: "Unit testing with Spring"
date: 2017-03-15
tags: unit-test spring
---

I'd like my code to be testable. Preferably with unit tests, which is much faster, than all the other types of tests.

Spring framework is good as it allows to get something working very simple and very fast. Consider

~~~
    @RestController @RequestMapping("/important-method")
    public class Controller extends BaseController {
    }
~~~
In like 2 minutes we get ourselves a working web service. Now we need to get some object, which would do something useful. Couple of more minutes and we have it like a charm

~~~
    @Configuration @ComponentScan
    public class Configuration {
        @Bean @Autowired
        public UsefulObject usefulObject(AnotherOne anotherOne) {
            return new UsefulObject(anotherOne);
        }
        @Bean
        public AnotherOne anotherOne() {
            return new AnotherOne();
        }
    }

    @RestController @RequestMapping("/important-method")
    public class Controller extends BaseController {

        @Autowired
        private UsefulObject usefulObject;
    }
~~~
Kinda neat, huh? In literally few minutes we are playing with *working* application.

But what about testing this? Unfortunately with this setup it's not that good, because in order to test `Controller`, we need to actually start the application. Spring helps with that, too, but it makes us to start it, which takes couple of seconds. Might not be much, but with true unit tests it's time to run may be a hundred of them.

So, what can we do to make it unit-testable. It's very simple and requires to use @Autowired only on constructors of that Controller.

~~~
    @RestController @RequestMapping("/important-method")
    public class Controller extends BaseController {
        private final UsefulObject usefulObject;

        @Autowired
        public Controller(UsefulObject usefulObject) {
            this.usefulObject = usefulObject;
        }
    }
~~~

Now, it's not required to start the app and we can create Controller in unit test with real UsefulObject or any kind of test double.

