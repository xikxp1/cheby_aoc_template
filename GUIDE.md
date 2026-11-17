# Solving puzzles in Cheby

This guide shows the patterns that Advent of Code puzzles need most often, written in Cheby: reading the input, repeating a step, counting, remembering results and searching. Every snippet compiles and runs with this template. [README.md](README.md) covers the commands.

The snippets leave out their imports. A name such as `list::map` needs `import std::list`, and `grid::parse` needs `import aoc::grid`.

## Read the input with `int::find_all`, `lines` and `split`

Most inputs are numbers with punctuation between them. `int::find_all` returns every integer of a text and skips the rest, so a table of numbers takes two calls:

```cheby
fn rows(input: String) -> List<List<Int>> {
  input
  |> string::lines
  |> list::map(_, int::find_all)
}
```

`rows("3   4\n4   -3\n")` gives `[[3, 4], [4, -3]]`. A `-` is a sign unless it follows a digit, so the range `2-4` reads as `[2, 4]`.

Other shapes need one of these:

| Input shape                     | Function                                                   |
| ------------------------------- | ---------------------------------------------------------- |
| Groups separated by blank lines | `blocks`, below                                            |
| One digit per character         | `digits`, below                                            |
| `name -> a, b`                  | `rule`, below                                              |
| A map drawn with characters     | `grid::parse(input)`, see [Search](#search-with-aocsearch) |

```cheby
fn blocks(input: String) -> List<List<String>> {
  input
  |> string::trim
  |> string::split(_, "\n\n")
  |> list::map(_, string::lines)
}

fn digits(line: String) -> List<Int> {
  line
  |> string::to_code_points
  |> list::filter_map(_, fn(c) { result::to_option(int::parse(c)) })
}

fn rule(line: String) -> Option<(String, List<String>)> {
  use (name, targets) <- option::map(string::split_once(line, " -> "))
  (name, string::split(targets, ", "))
}
```

`blocks("a\nb\n\nc\n")` gives `[["a", "b"], ["c"]]`, `digits("a1b2c3")` gives `[1, 2, 3]`, and `rule("in -> a, b")` gives `Some(("in", ["a", "b"]))`. The `trim` in `blocks` guards against a blank line at the end of the input, which would otherwise add an empty group.

## Repeat with a fold, `function::iterate` or a local function

Cheby has no loops. Choose the replacement by what repeats:

| What repeats                    | Use                                          |
| ------------------------------- | -------------------------------------------- |
| One step for each element       | `list::fold`                                 |
| A fixed number of rounds        | `function::iterate(state, 100, round)`       |
| Rounds until nothing changes    | `function::fixpoint(state, round)`           |
| Steps until a condition is true | a local `fn` that calls itself as its result |

A fold carries one value from element to element:

```cheby
fn fuel(masses: List<Int>) -> Int {
  list::fold(masses, 0, fn(total, mass) { total + mass / 3 - 2 })
}
```

A local `fn` sees the names around it and carries its state in its parameters. This one walks a list of changes again and again until a running total repeats:

```cheby
fn first_repeat(changes: List<Int>) -> Int {
  fn walk(left: List<Int>, total: Int, seen: set::Set<Int>) -> Int {
    case left {
      [] => walk(changes, total, seen)
      [change, ..rest] => {
        let sum = total + change
        case set::contains(seen, sum) {
          True => sum
          False => walk(rest, sum, set::insert(seen, sum))
        }
      }
    }
  }
  walk(changes, 0, set::from_list([0]))
}
```

`first_repeat([3, 3, 4, -2, -4])` gives `10`.

When the call to itself is the last thing the function does, as in `walk`, it uses no stack: a function of this shape counted to 200 million in 0.4 s. Name the helper `step` or `walk`, not `go`, which is a reserved word.

## Count groups when a population grows

When every item of a list turns into several each round, the list soon holds millions of items. Keep a map from each kind of item to how many there are, and step the map.

Take cells that age: a cell of age 0 becomes a cell of age 2 and makes a new cell of age 4, and any other cell gets one day younger.

```cheby
fn cells_after(ages: List<Int>, days: Int) -> Int {
  ages
  |> map::frequencies
  |> function::iterate(_, days, day)
  |> map::values
  |> list::sum(_, 0)
}

fn day(counts: map::Map<Int, Int>) -> map::Map<Int, Int> {
  map::fold(counts, map::new(), fn(next, age, n) {
    case age {
      0 => next
        |> add(_, 2, n)
        |> add(_, 4, n)
      _ => add(next, age - 1, n)
    }
  })
}

fn add(counts: map::Map<Int, Int>, age: Int, n: Int) -> map::Map<Int, Int> {
  map::upsert(counts, age, fn(old) { option::unwrap_or(old, 0) + n })
}
```

The map never holds more than five ages, whatever the number of cells. From one cell, 100 days give 66,869,517 cells in 0.1 ms. A version that keeps every cell in a list takes 82 ms for the 55,877 cells of day 60, and day 100 would need a list of 66 million.

This works whenever the order of the items does not change the answer.

## Remember results by passing a map through the calls

A recursion that meets the same question many times needs a cache. Cheby has no mutable state, so the cache is a map that goes into every call and comes back with the answer.

This function counts the ways to write `text` as pieces put end to end:

```cheby
fn ways(text: String, pieces: List<String>) -> Int {
  let (count, _) = ways_cached(text, pieces, map::new())
  count
}

fn ways_cached(
  text: String,
  pieces: List<String>,
  cache: map::Map<String, Int>,
) -> (Int, map::Map<String, Int>) {
  guard text != "" else {
    (1, cache)
  }
  case map::get(cache, text) {
    Some(known) => (known, cache)
    None => {
      let (total, filled) = list::fold(pieces, (0, cache), fn((sum, seen), piece) {
        case string::starts_with(text, piece) {
          True => {
            let (n, after) = ways_cached(string::remove_prefix(text, piece), pieces, seen)
            (sum + n, after)
          }
          False => (sum, seen)
        }
      })
      (total, map::insert(filled, text, total))
    }
  }
}
```

The cache appears in three places:

1. **The lookup** at the top returns a known answer with the cache unchanged.
2. **The fold** hands the cache from one recursive call to the next, so the second call sees what the first one learned.
3. **The insert** at the end stores the new answer before returning.

With the pieces `a`, `b`, `ab` and `ba`, `ways("abab", pieces)` gives `5`. Sixty letters of `abab…` have 2,504,730,781,961 ways, which the cached function counts in under 0.1 ms and a plain recursion would count one by one.

### Fill a table instead when the questions have an order

Here the answer for a suffix needs only the answers for shorter suffixes. A fold can then fill a table from the end of the text to its start, with no recursion and no cache to pass around:

```cheby
fn ways_table(text: String, pieces: List<String>) -> Int {
  let end = string::length(text)
  let starts = list::reverse(list::range(0, end))
  let table = list::fold(starts, map::from_list([(end, 1)]), fn(table, start) {
    let rest = string::slice(text, start, end)
    let count = pieces
      |> list::filter(_, string::starts_with(rest, _))
      |> list::map(_, fn(piece) { ways_at(table, start + string::length(piece)) })
      |> list::sum(_, 0)
    map::insert(table, start, count)
  })
  ways_at(table, 0)
}

fn ways_at(table: map::Map<Int, Int>, start: Int) -> Int {
  option::unwrap_or(map::get(table, start), 0)
}
```

`table` maps a start position to the number of ways to write the text from there. Use the cache when the order of the questions is hard to see, and the table when it is plain.

## Search with `aoc::search`

`aoc::search` needs two things from you: a start state and a function that lists the states one step away. A state can be any value, such as a position, or a tuple of a position and a direction.

**Fewest steps.** `search::bfs_to` returns the number of steps to the nearest state that passes a test. Here the states are the open cells of a maze:

```cheby
fn shortest(input: String) -> Option<Int> {
  let maze = grid::parse(input)
  use start <- option::then(grid::find(maze, "S"))
  search::bfs_to(
    start,
    fn(at) {
      at
      |> grid::neighbours4
      |> list::filter(_, fn(to) {
        case grid::get(maze, to) {
          Some("#") | None => False
          Some(_) => True
        }
      })
    },
    fn(at) { grid::get(maze, at) == Some("E") },
  )
}
```

`shortest("S.#\n..#\n#.E\n")` gives `Some(4)`, and a maze with no way through gives `None`. `search::bfs` takes no test and returns the step count of every state it reaches.

**Cheapest path.** `search::dijkstra` takes steps with a cost each. Here entering a cell costs its digit:

```cheby
fn cheapest(input: String) -> Option<Int> {
  let costs = grid::parse(input)
  let goal = (costs.width - 1, costs.height - 1)
  search::dijkstra(
    (0, 0),
    fn(at) {
      at
      |> grid::neighbours4
      |> list::filter_map(_, fn(to) {
        use cell <- option::then(grid::get(costs, to))
        use cost <- option::map(result::to_option(int::parse(cell)))
        (to, cost)
      })
    },
    fn(at) { at == goal },
  )
}
```

`cheapest("19\n11\n")` gives `Some(2)`: down, then right. `search::dijkstra_all` returns the cost of every state.

**A billion rounds.** When a puzzle asks for the state after a huge number of rounds, the states repeat. `search::state_after(start, round, 1_000_000_000)` finds the cycle and skips over it, and `search::find_cycle` returns where the cycle starts and how long it is.

## Check these first when a part is slow

Each run prints the time of each part. When one is too slow, look for these causes, most common first:

- **A list used as a set.** `list::contains` reads the whole list. Looking up 20,000 numbers took 7.3 s in a list of 20,000 and 3 ms in a `set::Set`.
- **Items kept one by one** that could be counted in groups, as in [Count groups](#count-groups-when-a-population-grows).
- **The same question answered again and again**, which a cache fixes, as in [Remember results](#remember-results-by-passing-a-map-through-the-calls).
- **A sort in every round** to find the smallest item. `heap::new_min()` with `heap::push` and `heap::pop` keeps the smallest on top.
- **A state larger than the question needs.** A search visits each distinct state once, so a state that carries the whole path visits far more states than one that carries a position.

`cheby run` has one speed. There is no optimized build to switch to yet.

## Look inside a pipeline with `io::debug`

`io::debug` prints a value to standard error and returns it, so it fits between two stages of a pipeline without changing the result. `io::debug_as` adds a label:

```cheby
let total = [1, 2, 3]
  |> list::map(_, fn(n) { n * n })
  |> io::debug
  |> list::sum(_, 0)
  |> io::debug_as(_, "total")
```

This prints `[1, 4, 9]` and `total: 14`. For a grid, `io::println(grid::draw(g))` prints it the way the puzzle draws it.

Put the puzzle's example in a `test` block before you run the real input. `cheby test --module aoc::day05` then tells you whether a change still solves it.

## Habits from other languages that Cheby rejects

Each of these is a compile error, and this table saves you the round trip:

| Habit                          | Write this instead                                               |
| ------------------------------ | ---------------------------------------------------------------- |
| `let total = total + 1`        | A new name: `let next = total + 1`. A name is bound once.        |
| `pair.0`                       | A pattern: `let (first, second) = pair`                          |
| `xs \|> list::map(double)`     | A hole for the piped value: `xs \|> list::map(_, double)`        |
| `"{int::show(n)}"`             | A name in the braces: `"{n}"`, after `let n = …` if it is a call |
| A helper named `go`            | `step` or `walk`; `go` is reserved                               |
| `for`, `while`, `if`, `return` | A fold or recursion, `case`, and `guard … else { … }`            |

One habit fails at run time instead: an `Int` holds 64 bits, and a result that does not fit stops the program with `integer overflow`. It never wraps around silently, so an answer that prints is not a wrapped one.
