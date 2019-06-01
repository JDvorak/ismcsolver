# ismcsolver [![Build Status][travis-badge]][travis] [![Build Status][appveyor-badge]][appveyor] [![codecov][codecov-badge]][codecov]

ismcsolver is a header-only C++14 library providing [information set Monte Carlo tree search][ISMCTS] (ISMCTS) algorithms. Monte Carlo tree search is a (game) AI decision making algorithm noted for its applicability to many different games of perfect information, requiring no domain-specific knowledge and only little information about a game's state. ISMCTS is an elegant extension of this technique to imperfect information games, where not all information is visible to all players, possibly combined with factors of randomness.

Strengths of the algorithm include being *anytime:* it can be given as much or as little CPU time as is available or desired, more time simply yielding more information on which to base the decision. Additionally it uses the available budget efficiently by only building a single information tree: identical moves from different possible game states all map to the same node, making it possible to find moves that are likely to be good in a variety of circumstances.

This code was originally part of a Qt game I wrote to explore game AI. I eventually wanted to test it with a simpler game, so I extracted it and also decided to port it to standard C++ to make it more useful any future projects. It now provides class templates that allow applying the algorithm to a generic game using a generic type of move.

[ISMCTS]: https://pure.york.ac.uk/portal/files/13014166/CowlingPowleyWhitehouse2012.pdf
[travis-badge]: https://travis-ci.org/sfranzen/ismcsolver.svg?branch=master
[travis]: https://travis-ci.org/sfranzen/ismcsolver
[appveyor-badge]: https://ci.appveyor.com/api/projects/status/3rbi2an9u3t06029?svg=true
[appveyor]: https://ci.appveyor.com/project/sfranzen/ismcsolver
[codecov-badge]: https://codecov.io/gh/sfranzen/ismcsolver/branch/master/graph/badge.svg
[codecov]: https://codecov.io/gh/sfranzen/ismcsolver

## Installation
Due to the nature of header-only libraries, no installation is technically necessary. You can use the headers in three ways:
* Using any build system:
    Simply copy the `include` folder to your project and configure your own build. In this case, you must ensure that build targets using a solver class are linked to an implementation of `<thread>`, e.g. by specifying `-lpthread`;
* Using CMake:
    * Clone this repository into a subdirectory of your project, e.g. ext/ismcsolver, and add it to your main CMakeLists.txt:
        ```cmake
        add_subdirectory(ext/ismcsolver)
        ```
    * Install the headers to a system location and use `find_package()`:
        ```cmake
        find_package(ismcsolver REQUIRED)
        ```
    In either case you can then link your targets as follows, which will automatically propagate compilation and linking options:
    ```cmake
    add_executable(foo foo.cpp)
    target_link_libraries(foo ismcsolver)
    ```

## Usage
This section only gives a few short examples of using the library. For a full reference to the class templates, generated by doxygen, please visit the [GitHub Page][docs] of this repository.

First and foremost, your game (or engine) class should implement the abstract interface [`ISMCTS::Game<Move>`][game], replacing the template parameter `Move` with the data type (or class) representing a player's move. The library provides two solver class templates, [`ISMCTS::SOSolver<Move>`][SO] and [`ISMCTS::MOSolver<Move>`][MO]. Their differences are explained [below](#features), but they should be instantiated with the same `Move` type to obtain an object that can select moves for the AI player(s), e.g.
```cpp
#include <ismcts/sosolver.h>
// ...
void MyGame::doAIMove()
{
    ISMCTS::SOSolver<MyMove> solver;
    this->doMove(solver(*this));
}
```
It may be desirable to inspect the information tree in order to tune the solver parameters to a particular game. After all, the algorithm is heavily influenced by, among other things, the number of moves available and the magnitude of the result values returned by the game. Solvers provide access to the tree(s) built during the previous search through the `currentTrees()` method. The resulting structure depends on the type of solver, but will contain instances of [`ISMCTS::Node<Move>`][node] representing the root node of an individual tree, which can be manipulated using the available methods. String information about a single node can be obtained from `operator std::string()` or written to an output stream with `operator<<`. An entire (sub)tree can also be output using a node's `treeToString()` method.

As an example, consider the perfect information [m-n-k game], for which this algorithm is certainly not the best approach. A game state with a winning move available to player 0 may look like this:
```
  1  0  0
  1  0 -1
 -1  1 -1
```
where the moves are numbered 0 through 8 and -1 represents available moves. A tree search of 1000 iterations from this state might produce the following result:
```
[M:0 by -1, S/V/A: 0.0/1000/1]
| [M:6 by 0, S/V/A: 988.0/988/998]
| [M:8 by 0, S/V/A: 1.0/5/998]
| | [M:6 by 1, S/V/A: 3.0/3/3]
| | [M:5 by 1, S/V/A: 0.0/1/3]
| [M:5 by 0, S/V/A: 2.0/7/998]
| | [M:8 by 1, S/V/A: 0.0/1/5]
| | [M:6 by 1, S/V/A: 5.0/5/5]
```
indicating that move 6 was the most visited and would therefore be selected.

A simple card game (Knockout Whist) is also included [here][whist], based on an [example Python code][py]. The main purpose of the included games is to test the compilation and functioning of the library code, but they also serve as example implementations of the `ISMCTS::Game` interface.

[docs]: https://sfranzen.github.io/ismcsolver/
[game]: include/ismcts/game.h
[SO]: include/ismcts/sosolver.h
[MO]: include/ismcts/mosolver.h
[node]: include/ismcts/node.h
[m-n-k game]: test/common/mnkgame.cpp
[whist]: test/common/knockoutwhist.cpp
[py]: https://gist.github.com/kjlubick/8ea239ede6a026a61f4d

## Type requirements
The type `Move` specified for the game and sequential solver templates must be a *[TrivialType]* that is *[EqualityComparable]*. Solvers with root parallelisation use a `std::map` to evaluate visit counts, so the `Move` must additionally be *[LessThanComparable]*.

[TrivialType]: https://en.cppreference.com/w/cpp/named_req/TrivialType
[EqualityComparable]: https://en.cppreference.com/w/cpp/named_req/EqualityComparable
[LessThanComparable]: https://en.cppreference.com/w/cpp/named_req/LessThanComparable

## About random numbers
Game implementations will need a source of random numbers. Because `std::rand` does not guarantee a good quality sequence and may not be thread safe, modern code should use the `<random>` header. The objects representing generator engines are large, so the following is a good way of providing one to your class:
```cpp
#include <random>
// definition as member of MyGame, could be const or static
std::mt19937 &MyGame::prng()
{
    static thread_local std::mt19937 prng {std::random_device{}()};
    return prng;
}
```
This sets up one pseudorandom Mersenne Twister engine per thread, seeded with random numbers from a non-deterministic source if available. It can be used for shuffling, e.g. `std::shuffle(cards.begin(), cards.end(), prng());`. Single numbers on (closed) intervals should be generated using one of the available distributions:
```cpp
std::uniform_int_distribution<> singleDie {1, 6};
auto result = singleDie(prng());
```

## Features
The basic algorithm can be applied, modified and executed in different ways. This section lists what is currently implemented.

### SO- and MO-ISMCTS
These are two variants of the algorithm, respectively Single-Observer and Multiple-Observer. The former is implemented in `ISMCTS::SOSolver` and, as its name implies, observes the game from only one perspective: that of the player conducting the search. It only builds a tree for that player and treats all opponent moves as fully observable. While that is indeed the case in many games, some additionally feature actions that are hidden from the other players. MO-ISMCTS, implemented in `ISMCTS::MOSolver`, is intended for such games; it builds a tree for each player to model the presence of this extra hidden information.

### Multithreading
Several approaches to parallel tree searching exist, see e.g. [Parallelization of Information Set Monte Carlo Tree Search][par] by Nick Sephton and the authors of ISMCTS.

These are implemented using `std::async` and can be used by providing the optional second `ExecutionPolicy` parameter to either class template. For example, the following instantiates a RootParallel `SOSolver` for some game where `int` represents a move:

```cpp
ISMCTS::SOSolver<int, ISMCTS::RootParallel> solver;
```

Template specialisations are provided for three policies, defined in [execution.h]:
* `ISMCTS::Sequential`: no multithreading, the default;
* `ISMCTS::RootParallel`: each system thread searches a separate tree structure. The statistics from the root of each tree are then combined to find the overall best move. This method is very fast as it avoids synchronisation issues and the overhead for finding a move is limited to a single layer of nodes per tree.
* `ISMCTS::TreeParallel`: the threads share a single tree structure. This is slightly slower, because threads may compete for access to nodes. However, the difference is typically small because the nodes quickly outnumber the threads.

[execution.h]: include/ismcts/execution.h
[par]: https://www-users.cs.york.ac.uk/~nsephton/papers/wcci2014-ismcts-parallelization.pdf

### Time-limited execution
Each solver type has two constructors; one that sets the search operator to iterate a fixed number of times, the other instead letting it search for a fixed length of time (a `std::chrono::duration<double>`). Both the mode of operation and the length of the search can be changed after instantiation. Durations can be conveniently created by including [`std::chrono`][chrono] and using the `std::chrono_literals`, e.g.:

```cpp
using namespace std::chrono_literals;
ISMCTS::SOSolver<int> solver {5ms};
```

[chrono]: https://en.cppreference.com/w/cpp/header/chrono

## License
This project is licensed under the MIT License, see the [LICENSE](LICENSE) file for details.
