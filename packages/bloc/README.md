<img src="https://raw.githubusercontent.com/felangel/bloc/master/docs/assets/bloc_logo_full.png" height="60" alt="Bloc Package" />

[![Build Status](https://travis-ci.org/felangel/bloc.svg?branch=master)](https://travis-ci.org/felangel/bloc)
[![codecov](https://codecov.io/gh/felangel/Bloc/branch/master/graph/badge.svg)](https://codecov.io/gh/felangel/bloc)
[![Pub](https://img.shields.io/pub/v/bloc.svg)](https://pub.dartlang.org/packages/bloc)
[![Awesome Flutter](https://img.shields.io/badge/Awesome-Flutter-turquoise.svg?longCache=true)](https://github.com/Solido/awesome-flutter)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Gitter](https://img.shields.io/badge/gitter-bloc-yellow.svg)](https://gitter.im/bloc_package/Lobby)

---

A dart package that helps implement the [BLoC pattern](https://www.youtube.com/watch?v=fahC3ky_zW0).

This package is built to work with [RxDart.dart](https://pub.dartlang.org/packages/rxdart) 0.18.1+.

## Overview

<img src="https://raw.githubusercontent.com/felangel/bloc/master/docs/assets/bloc_architecture.png" alt="Bloc Architecture" />

The goal of this package is to make it easy to implement the `BLoC` Design Pattern (Business Logic Component).

This design pattern helps to separate _presentation_ from _business logic_. Following the BLoC pattern facilitates testability and reusability. This package abstracts reactive aspects of the pattern allowing developers to focus on converting events into states.

## Glossary

**Events** are the input to a Bloc. They are commonly UI events such as button presses. `Events` are `dispatched` and then converted to `States`.

**States** are the output of a Bloc. Presentation components can listen to the stream of states and redraw portions of themselves based on the given state (see `BlocBuilder` for more details).

**Transitions** occur when an `Event` is `dispatched` after `mapEventToState` has been called but before the `Bloc`'s state has been updated. A `Transition` consists of the currentState, the event which was dispatched, and the nextState.

**BlocSupervisor** oversees `Bloc`s and delegates to `BlocDelegate`.

**BlocDelegate** handles events from all `Bloc`s which are delegated by the `BlocSupervisor`. Can be used to observe all `Bloc` `Transition`s. **It is a great way to handle logging/analytics universally**.

## Bloc Interface

**initialState** is the state before any events have been processed (before `mapEventToState` has ever been called). `initialState` **must be implemented**.

**mapEventToState** is a method that **must be implemented** when a class extends `Bloc`. The function takes two arguments: state and event. `mapEventToState` is called whenever an event is `dispatched` by the presentation layer. `mapEventToState` must convert that event, along with the current state, into a new state and return the new state in the form of a `Stream` which is consumed by the presentation layer.

**dispatch** is a method that takes an `event` and triggers `mapEventToState`. `dispatch` may be called from the presentation layer or from within the Bloc (see examples) and notifies the Bloc of a new `event`.

**transform** is a method that can be overridden to transform the `Stream<Event>` before `mapEventToState` is called. This allows for operations like `distinct()` and `debounce()` to be used.

**onTransition** is a method that can be overridden to handle whenever a `Transition` occurs. A `Transition` occurs when a new `Event` is dispatched and `mapEventToState` is called. `onTransition` is called before a `Bloc`'s state has been updated. **It is a great place to add bloc-specific logging/analytics**.

## BlocDelegate Interface

**onTransition** is a method that can be implemented to handle whenever a `Transition` occurs from **any** `Bloc`. **It is a great place to add universal logging/analytics**.

## Usage

For simplicity we can create a `Bloc` that always returns a stream of static strings in response to any event. That would look something like:

```dart
class SimpleBloc extends Bloc<dynamic, String> {
  @override
  String get initialState => '';

  @override
  Stream<String> mapEventToState(String currentState, dynamic event) async* {
    yield 'data';
  }
}
```

That isn't a very realistic use-case so let's take something more practical like a login flow.

We're going to need to define what our different `LoginStates` are going to be.
For simplicity, let's say we only have 4 states:

- `initial`
- `loading`
- `failure`
- `success`

```dart
class LoginState {
  final bool isLoading;
  final bool isLoginButtonEnabled;
  final String error;
  final String token;

  const LoginState({
    @required this.isLoading,
    @required this.isLoginButtonEnabled,
    @required this.error,
    @required this.token,
  });

  factory LoginState.initial() {
    return LoginState(
      isLoading: false,
      isLoginButtonEnabled: true,
      error: '',
      token: '',
    );
  }

  factory LoginState.loading() {
    return LoginState(
      isLoading: true,
      isLoginButtonEnabled: false,
      error: '',
      token: '',
    );
  }

  factory LoginState.failure(String error) {
    return LoginState(
      isLoading: false,
      isLoginButtonEnabled: true,
      error: error,
      token: '',
    );
  }

  factory LoginState.success(String token) {
    return LoginState(
      isLoading: false,
      isLoginButtonEnabled: true,
      error: '',
      token: token,
    );
  }
}
```

Next we need to define the different events that our Bloc will respond to. Again, for simplicity, let's say there is just a single event we will handle: `LoginButtonPressed`.

```dart
abstract class LoginEvent {}

class LoginButtonPressed extends LoginEvent {
  final String username;
  final String password;

  LoginButtonPressed({@required this.username, @required this.password});
}
```

Now that we've identified our `states` and `events`, our `LoginBloc` should look something like:

```dart
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginState get initialState => LoginState.initial();

  void onLoginButtonPressed({String username, String password}) {
    dispatch(
      LoginButtonPressed(
        username: username,
        password: password,
      ),
    );
  }

  @override
  Stream<LoginState> mapEventToState(LoginState currentState, LoginEvent event) async* {
    if (event is LoginButtonPressed) {
      yield LoginState.loading();

      try {
        final token = await _authenticate(event.username, event.password);
        yield LoginState.success(token);
      } catch (error) {
        yield LoginState.failure(error.toString());
      }
    }
  }
}
```

As our app grows and relies on multiple `Bloc`s, it becomes useful to see the `Transitions` for all `Bloc`s. This can easily be achieved by implementing a `BlocDelegate`.

```dart
class SimpleBlocDelegate implements BlocDelegate {
  @override
  void onTransition(Transition transition) {
    print(transition.toString());
  }
}
```

Now that we have our `SimpleBlocDelegate`, we just need to tell the `BlocSupervisor` to use our delegate in our `main.dart`.

```dart
void main() {
  BlocSupervisor().delegate = SimpleBlocDelegate();
}
```

At this point, all `Bloc` `Transitions` will be reported to the `SimpleBlocDelegate`.

## Dart Versions

- Dart 2: >= 2.0.0

## Examples

- [Simple Counter Example](https://github.com/felangel/Bloc/tree/master/packages/bloc/example) - an example of how to create a `CounterBloc` (pure dart)
- [Login Flow Example](https://github.com/felangel/Bloc/tree/master/examples/flutter_login) - an example of how to use the `bloc` and `flutter_bloc` packages to implement a Login Flow.

### Contributors

- [Felix Angelov](https://github.com/felangel)
