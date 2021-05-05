# Software Engineering For Embedded Applications

## Week 8 (May 12, 2021): Events and Finite State Machines

---

Last week we built a simple process manager that included an abstract base class, `Process`, from which users can derive their own processes. We also defined a `Channel` class that enabled processes to send streams of values to each other.

<img src='https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/elma-structure.png' width=70%>

This week we will

- Upgrade the `Channel` class to enable sending _JSON_ data, instead of just `double` values.
- Add a new inter-process communication `Event` class.
- Add a set of classes for defining _Finite State Machines_.

The resulting structure looks like the above. Orange indicates classes that we will define today.

# Elma Changes

Please restart your docker container with something like

```bash
docker pull klavins/elma:latest
```

which should download the latest version from DockerHub, before doing

```bash
docker run -v $PWD:/source -it klavins/elma:latest bash
```

# JSON

The _Javascript Object Notation_ or [JSON](http://www.json.org/) standard is a file and data format used almost ubiquitously in networked systems. It has become the standard way to represent arbitrary data types in a structured way, and to allow applications to communicate without much overhead in defining data structures and types. It is even used in some of the most popular databases, such as MongoDB, as the fundamental data structure for defining documents.

Some examples of where JSON is used include

- [Vehicle APIs](https://www.scip.ch/en/?labs.20180405)
- [Industrial Automation](https://openautomationsoftware.com/)
- [Databases](https://www.mongodb.com/)
- [Google Maps](https://www.sitepoint.com/google-maps-json-file/)
- [NOAA Data Services](https://www.ncdc.noaa.gov/access-data-service/api/v1/data?dataset=global-summary-of-the-year&dataTypes=DP01,DP05,DP10,DSND,DSNW,DT00,DT32,DX32,DX70,DX90,SNOW,PRCP&stations=ASN00084027&startDate=1952-01-01&endDate=1970-12-31&includeAttributes=true&format=json)

# Data Representation in JSON

Example:

```json
{
  "ta": [
    {
      "first": "Trung",
      "last": "Le"
    }
  ],
  "weeks": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
  "version": 3.0
}
```

- **_Atomic_** values
  - strings delimited by double quotes
  - numbers, with or without decimals
  - true and false
  - null
- **_Arrays_** of values atomic values, objects and/or otjer arrays, delimited by [ and ].
- **_Objects_** with key value pairs, where values are atomic, objects, or arrays

# Supporting JSON

The two key methods that one needs to support JSON in an application are

- _Serialization_, also knownn as _stringification_: Converting an application data structure into a JSON string.
- _Deserialization_, also known as _parsing_: Converting a JSON string into an application data structure.

For example, open the console on your web browser and type

```javascript
str = '{ "a": 1, "b": [2,3], "k": { "c": { "d": true } } }';
x = JSON.parse(str);
x.k.c.d; // true
JSON.stringify(x);
```

Note that if your string has a format error in it, you will get an error when you try to deserialize it:

```javascript
JSON.parse('{"x": [1,2}');
```

which results in the useful message:

```
VM282:1 Uncaught SyntaxError: Unexpected token } in JSON at position 10
    at JSON.parse (<anonymous>)
    at <anonymous>:1:6
```

# Adding JSON to your code (and to Elma)

Using the elma container, you can use the JSON library simply by including it.

```c++
#include <json/json.h>

using nlohmann::json;
```

# A JSON C++ Library

The JSON C++ library we will use is described in detail [here](https://github.com/nlohmann/json). For now, we will use a subset of the library's capabilities, mainly defining and accessing JSON values.

To defining a JSON value, you can simply declare it with the json type and assign it to a C++ value. For example,

```c++
json j = 3.14,
     k = "hello world",
     l = { 1,2,3 };
```

defines three JSON values with raw types double, string, and integer array. Note that since C++ does not use [ and ] to define arrays, we have to use { and }.

Since C++ does not have built in hash-like objects, to define an object, you have to use a special notation that does not look much like JSON. For example,

```c++
json j = { { "x", 0 }, { "y", 1 } }; // { "x": 0, "y": 1 }
```

# Nested JSON

If you want to define a nested object, you can write

```c++
json j = {
    {
        "ta", {
            {
                { "first", "Trung" },
                { "last", "Le" }
            }
        },
    },
    {
        "weeks", { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 }
    },
    {
        "version", 2.0
    }
};
```

# Serializing a String

You can serialize a string directly in a definition, as in

```c++
json x = R"('{ "a": 1, "b": [2,3], "k": { "c": { "d": true } } }')"_json;
```

This magical bit of code uses the C++ [raw string operator](http://www.cplusplus.com/forum/general/190123/). The `_json` bit at the end is using a [user defined literal](https://en.cppreference.com/w/cpp/language/user_literal) (defined in the JSON library).

# Accessing Parts of Objects

To access parts of a json object, you use notation such as

```c++
int a = x["a"]; // 1
bool d = x["k"]["c"]["d"]; // true
```

Note that the assignment operator `=` has been overloaded to convert any kind of json value into a C++ value (such as `int` and `bool` above). You will get a runtime error if the value cannot be converted. For example, if you do

```c++
double k = x["k"];
```

you will get

```
C++ exception with description "[json.exception.type_error.302] type must be number, but is object" thrown in the test body.
```

# Downsides of JSON

This brings up one of the downsides of using JSON. With C++ you normally cannot even compile code that tries to assign a value of one type to an incompatible value of another type. But with JSON you can. If you would like to be very explicit, you can instead do

```c++
double k = x["k"].get<double>();
```

which uses the templated `get` method of the library. However, this will still throw a runtime error, since the compiler can't keep track of what values `x` may take on without running the program.

# Upgrading Channels to use JSON

The JSON library is incredibly easy to use. To modify the `Channel` object, we literally replace the keyword `double` with the type `json`. In the header file:

```c++
class Channel {
    //
    public:
    Channel& send(json);
    json latest();
    json earliest();
    //
    private:
    deque<json> _queue;
    //
};
```

# Implementation

In the implementation file

```c++
Channel& Channel::send(json value) {
    _queue.push_front(value);
    while ( _queue.size() > capacity() ) {
        _queue.pop_back();
    }
    return *this;
}

json Channel::latest() {
    if ( _queue.size() == 0 ) {
        throw Exception("Tried to get the latest value in an empty channel.");
    }
    return _queue.front();
}

// etc
```

# Events

An _event_ is a discrete occurrence that happens at a specific point in time. For example, a touch sensor might go from off to on, or a user might press a button. Events often have data associated with them. For example, how hard was the button pressed? To capture an event formally, we define a new class:

```c++
class Event {

    public:

        Event(std::string name, json value) : _name(name), _value(value), _empty(false) {}
        Event(std::string name) : _name(name), _value(0), _empty(true) {}

    private:
    std::string _name;
    json _value;
    bool _empty; // used when no value is needed or provided

};
```

We will expand upon this definition later, but for now, an event is little more than a container for a JSON value.

# Event Methods

To use an event, processes need to be able to

- **emit** an event, giving it a name in the process. The result should be that the manager broadcasts the occurence of the event to any other processes that are watching for it.
- **watch** for events with a specific name, responding to them with user defined functions.

# Keeping Track of Events in the Manager

To the `Manager` we add a private data member

```c++
// manager.h
private:
map<string, vector<std::function<void(Event&)>>> event_handlers;
```

# Watching Events

Next we add a `watch` method to Manager and a wrapper for it to `Process`.

```c++
// manager.cc
Manager& Manager::watch(string event_name, std::function<void(Event&)> handler) {
    event_handlers[event_name].push_back(handler);
    return *this;
}

// process.cc
void Process::watch(string event_name, std::function<void(Event&)> handler) {
    _manager_ptr->watch(event_name, handler);
}
```

# Where You Should use Watch

Typically, processes should set up event handlers in their `init` methods. For example, you could do

```c++
// Cruise control process watching for desired speed events from the driver
void CruiseControl::init() {
    watch("desired speed", [this](Event& e) {
        desired_speed = e.value();
    });
}
```

This method would be used by a `CruiseControl` process to respond to an event changing its desired speed. Note that the lambda sent to the event captures `this`, so that the variable `desired_speed` that is private to the CruiseControl object can be accessed.

# Emitting an Event

To emit an event, we define a `Manager` method and a `Process` wrapper that searches the event handler list for handlers that correspond to the emitted event.

```c++
// Manager.cc
Manager& Manager::emit(const Event& event) {
    Event e = event; // make a copy so we can change propagation
    if ( event_handlers.find(event.name()) != event_handlers.end() ) {
        for ( auto handler : event_handlers[event.name()] ) {
            handler(e);
        }
    }
    return *this;
}

// Process.cc
void Process::emit(string event_name, const Event& event) {
    _manager_ptr->emit(event_name, event);
}
```

# Example Event Emission

A process would typically emit an event in its `start`, `update`, or `stop` method.

For example, suppose you wanted to simulate a driver with a `Driver` process.

```c++
void Driver::start() {
    emit(Event("desired speed", 40));
}
```

When this process is started, it will emit the event, which the cruise control process will respond to.

# Example

See `examples/driving.cc` for a worked out example. A plot of the velocity over time from that example is here:

<img src='https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/cruise-control-output.png' width=70%>

# Event Propagation

The way we currently have event manager defined, every handler that watches for events with a particular name will get run every time an event with that name is emitted. This may not be desired in some cases. For example, you may want that certain events take priority and prevent other handlers from being run. To get this behavior, we introduce _event propagation_.

# Changing Event Propagation

To the event class, we add a Boolean value to keep track of whether the event should be propagated.

```c++
// event.h
private:
bool _propagate;
```

as well as the methods:

```c++
// event.h
inline bool propagate() const { return _propagate; }
inline void stop_propagation() { _propagate = false; }
inline void reset() { _propagate = true; }
```

# Managing Propagation

In the manager, we can then prevent events from propagating if the-r `_propgate` instance variable is set to true as follows:

```c++
Manager& Manager::emit(const Event& event) {
    Event e = event; // new: make a copy so we can change propagation
    if ( event_handlers.find(event.name()) != event_handlers.end() ) {
        for ( auto handler : event_handlers[event.name()] ) {
            if ( e.propagate() ) { // new
                handler(e);
            }
        }
    }
    return *this;
}
```

# Using Event Propagation

This capability will become particularly useful in the next section on finite state machines.

# Finite State Machines

A finite state machine, or FSM, is a fundamental object in embedded systems. They consist of a set of states and a set of labelede transitions between states. Here is a simple example of a toggle switch.

<img src='https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/toggle-switch.png' width=70%>

There are two states, `Off` and `On`. The FSM moves from one state to the other, every time a 'switch' input is recieved.

# Microwave Example

Another example is a microwave oven controller, which is designed to accept user input and keep the user from doing something bad (like tuning on the microwave when the door is open).

<img src="https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/microwave.png" width="500"></image>

To implement FSMs in Elma, we will add three new classes: `State`, `Transition`, and `StateMachine`. The first is an abstract base class that users will override with their own state definitions. Transition is a container class that holds a source and desintation state and an event name. StateMachine will inherit from `Process` and will manager transitions.

# States

To represent states, we add the new class:

```c++
class State {

    friend class StateMachine;

    public:
    State();
    State(std::string name);

    inline std::string name() { return _name; }
    virtual void entry(Event& e) = 0;
    virtual void during() = 0;
    virtual void exit(Event& e) = 0;

    void emit(const Event& e);

    private:
    std::string _name;
    StateMachine * _state_machine_ptr;

};
```

# Transitions

To represent transitions, we add the new class:

```c++
class Transition {

    public:
    Transition(std::string event_name, State& from, State& to) :
        _from(from),
        _to(to),
        _event_name(event_name) {}

    inline State& from() const { return _from; }
    inline State& to() const { return _to; }
    inline string event_name() const { return _event_name; }

    private:
    State& _from;
    State& _to;
    string _event_name;

};
```

# State Machines

To represent state machines, we add the new class

```c++
class StateMachine : public Process {

    public:
    StateMachine(std::string name) : Process(name) {}
    StateMachine() : Process("unnamed state machine") {}

    StateMachine& set_initial(State& s);
    StateMachine& add_transition(std::string event_name, State& from, State& to);
    inline StateMachine& set_propagate(bool val) { _propagate = val; }

    State& current() { return *_current; }

    void init();
    void start();
    void update();
    void stop();

    private:
    vector<Transition> _transitions;
    State * _initial;
    State * _current;
    bool _propagate;

};
```

# State Machine Initialization

The class keeps track of the initial and current states as well as keeping a list of transitions. Since it inherits from `Process` and needs its `init`, `start`, `update`, and `stop` methods defined. The `init` method loops through all transitions and watches for events that trigger them.

```c++
void StateMachine::init() {
    for (auto transition : _transitions ) {
        watch(transition.event_name(), [this, transition](Event& e) {
            if ( _current->id() == transition.from().id() ) {
                _current->exit(e);
                _current = &transition.to();
                _current->entry(e);
                if ( !_propagate ) { // StateMachine has a wrapper flag for
                                     // whether it should propagate
                    e.stop_propagation();
                }
            }
        });
    }
}
```

# State Machine Starting and Updating

The start method sets the current state to the initial state

```c++
void StateMachine::start() {
    _current = _initial;
    _current->entry(Event("start"));
}
```

and the update method calls the current state's during method.

```c++
void StateMachine::update() {
    _current->during();
}
```

# Example: Binary Counter

See the `toggle-switch/` directory for how to implement the toggle switch.

<img src='https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/toggle-switch.png' width=40%>

# Example: Microwave Oven

See the `microwave/` directory for how to implement the microwave oven.

<img src="https://raw.githubusercontent.com/klavins/EEP520-W20/master/week_7/images/microwave.png" width="500"></image>

# Documentation with Doxygen

The documentation for Elma is current at [https://klavins.github.io/ECEP520/index.html](https://klavins.github.io/ECEP520/index.html). It is automatically generated by first running Doxygen with

```bash
make docs
```

and then copying the contents of the resulting `html` directory to `ECEP520/docs`. Github has a feature called _Github Pages_ that allows you to serve web pages from a repository. To set it up, I went to the ECEP520 repository, clicked settings, scrolled down to _Github Pages_, and chose `/docs` as the source directory.

# Documenting a Class

To document a class, you add comments to the code with the special syntax

```c++
//! My new class

//! Some details about my class
class MyClass {

};
```

# Documenting a Method

To document a method, you do something like

```c++
//! Compute something important
//! \param x A description of the first argument
//! \param s A description of the second argument
//! \return The value of the important thing computed
int f(int x, std::string s);
```

# Adding a Code Example

To add a code example to a comment, do

```c++
//! @code
//! int x = 1;
//! @endcode
```

# For more examples

See, for example, the comments the [elma code base](https://github.com/klavinslab/elma) which compiles to look like [this](http://klavinslab.org/elma/).
