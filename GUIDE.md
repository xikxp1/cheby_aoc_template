# Solving puzzles in Cheby

This guide shows the patterns that Advent of Code puzzles need most often, written in Cheby: reading the input, repeating a step, counting, remembering results, indexing lists, merging ranges, working with bits and searching. Every snippet compiles and runs with this template. [README.md](README.md) covers the commands.

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
| One digit per character         | `int::digits(line)`, in `digit_rows` below                 |
| A letter, then a number: `L68`  | `string::first` and `string::drop_start`, in `turn` below  |
| `name -> a, b`                  | `rule`, below                                              |
| A map drawn with characters     | `grid::parse(input)`, see [Search](#search-with-aocsearch) |

```cheby
fn blocks(input: String) -> List<List<String>> {
  input
  |> string::lines
  |> list::split_when(_, fn(line) { line == "" })
}

fn digit_rows(input: String) -> List<List<Int>> {
  input
  |> string::lines
  |> list::map(_, fn(line) { result::expect(int::digits(line), "not a line of digits") })
}

fn turn(line: String) -> (String, Int) {
  let letter = option::expect(string::first(line), "an empty line")
  let distance = line
    |> string::drop_start(_, 1)
    |> int::parse
    |> result::expect(_, "no number after the letter")
  (letter, distance)
}

fn rule(line: String) -> Option<(String, List<String>)> {
  use (name, targets) <- option::map(string::split_once(line, " -> "))
  (name, string::split(targets, ", "))
}
```

`blocks("a\nb\n\nc\n")` gives `[["a", "b"], ["c"]]`, `digit_rows("123\n405\n")` gives `[[1, 2, 3], [4, 0, 5]]`, `turn("L68")` gives `("L", 68)`, and `rule("in -> a, b")` gives `Some(("in", ["a", "b"]))`.

- `list::split_when` cuts a list at the elements that pass a test and drops the empty runs, so blank lines at the end of the input, or two in a row, add no empty group. It cuts any list, such as the columns of `list::transpose` at the blank ones.
- `int::digits` is strict: a line with any other character, a space or a `\r` included, gives `Err(Nil)`. `result::expect` then stops the program with its message and the line of the call, instead of dropping the character. For the digits of a number, `int::to_digits(907)` gives `[9, 0, 7]`.
- `string::first` and `string::last` give the first and last character, and `string::drop_start` and `string::drop_end` remove characters from that end. A pattern cannot match the start of a string, so these replace `"L" <> rest`.

## Parse once for both parts with `kit::run_parsed`

When both parts read the input the same way, parse it once. `kit::run_parsed` takes a `parse` function and gives its result to both parts. Its second argument is `True` on `--sample`, for puzzles whose sample asks for other numbers than your input, such as 10 links instead of 1000:

```cheby
type Puzzle {
  boxes: List<List<Int>>,
  links: Int,
}

fn parse(input: String, on_sample: Bool) -> Puzzle {
  let links = case on_sample {
    True => 10
    False => 1000
  }
  let boxes = input
    |> string::lines
    |> list::map(_, int::find_all)
  Puzzle { boxes, links }
}

fn main() -> Result<Nil, kit::Error> {
  kit::run_parsed(8, parse, part1, part2)
}
```

Here `part1` and `part2` each take a `Puzzle`. The run prints the parse's time on a line of its own, before the parts. A test calls `parse` itself, as in `part1(parse(sample, True))`.

When only the numbers differ and the parts parse the input in their own ways, `kit::run_with(8, part1, part2)` passes each part the text and the same `Bool`, as in `fn part1(input: String, on_sample: Bool) -> Int`.

## Repeat with a fold, `function::iterate` or a local function

Cheby has no loops. Choose the replacement by what repeats:

| What repeats                         | Use                                          |
| ------------------------------------ | -------------------------------------------- |
| One step for each element            | `list::fold`                                 |
| The same, keeping every step's value | `list::scan`                                 |
| A fixed number of rounds             | `function::iterate(state, 100, round)`       |
| Rounds until nothing changes         | `function::fixpoint(state, round)`           |
| Steps until a condition is true      | a local `fn` that calls itself as its result |

A fold carries one value from element to element:

```cheby
fn fuel(masses: List<Int>) -> Int {
  list::fold(masses, 0, fn(total, mass) { total + mass / 3 - 2 })
}
```

`list::scan` takes the same arguments as `list::fold` and returns the value after each element, without the starting one. Here it gives the running position of a dial, and `list::count` counts the times it stops at 0:

```cheby
fn stops_at_zero(turns: List<Int>) -> Int {
  turns
  |> list::scan(_, 50, int::add)
  |> list::count(_, fn(position) { position % 100 == 0 })
}
```

`stops_at_zero([-68, -30, 48, -5, 60, 45])` gives `2`: the positions are `[-18, -48, 0, -5, 55, 100]`. When the fold only adds or counts, `list::sum` and `list::count` say so directly, and so do `set::sum`, `set::count` and `map::count`. `set::sum` adds in sorted order, so it gives the same result on every run.

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
      |> list::map(_, fn(piece) { map::get_or(table, start + string::length(piece), 0) })
      |> list::sum(_, 0)
    map::insert(table, start, count)
  })
  map::get_or(table, 0, 0)
}
```

`table` maps a start position to the number of ways to write the text from there. `map::get_or` returns the value of a key, or its third argument when the map has no such key. Use the cache when the order of the questions is hard to see, and the table when it is plain.

## Index into a list

A `List` is a persistent vector, an RRB tree, not a linked list. `list::get`, `list::set`, `list::push`, `list::prepend`, `list::append` and `list::slice` each take O(log n), and `push` and `set` on a list that no other value holds update it in place, so a `push` in a fold costs amortized O(1). Reading by index is fine, and a `List<List<Int>>` is a good table.

A table of prefix sums answers the sum of any stretch with two reads:

```cheby
fn prefix_sums(xs: List<Int>) -> List<Int> {
  [0, ..list::scan(xs, 0, int::add)]
}

fn range_sum(sums: List<Int>, from: Int, to: Int) -> Int {
  let high = option::expect(list::get(sums, to), "to is past the end")
  let low = option::expect(list::get(sums, from), "from is past the end")
  high - low
}
```

`range_sum(prefix_sums([3, 1, 4, 1, 5]), 1, 4)` gives `6`, the sum of the elements at 1, 2 and 3. `list::get` returns `None` outside the list. `option::expect` unwraps the `Some` and stops the program with its message and the line of the call on `None`, which suits an index that cannot be out of range. When a missing element has a sensible value, `list::get_or(xs, index, 0)` returns that value instead, as `map::get_or` does for a key: `map::get_or(cells, at, ".")` reads every cell outside a grid as empty.

`list::max_index` and `list::min_index` give the index of the largest and smallest element, for a greedy choice. They break ties as `list::max` and `list::min` do: `max_index` gives the last of equal maxima, and `min_index` the first of equal minima. This function picks `count` digits of a bank, in order, to make the largest number. It needs the first of equal maxima, which leaves the most digits for later, so it negates the digits and asks `min_index`:

```cheby
fn largest(bank: List<Int>, count: Int) -> Int {
  fn pick(rest: List<Int>, left: Int, value: Int) -> Int {
    guard left > 0 else {
      value
    }
    let window = list::drop_last(rest, left - 1)
    let negated = list::map(window, fn(digit) { 0 - digit })
    let index = option::expect(list::min_index(negated), "too few digits")
    pick(list::drop(rest, index + 1), left - 1, value * 10 + list::get_or(window, index, 0))
  }
  pick(bank, count, 0)
}
```

`largest([8, 1, 1, 9, 1, 9], 2)` gives `99`, and `largest(int::to_digits(818_181_911_112_111), 12)` gives `888_911_112_111`. `list::drop_last` and `list::take_last` remove or keep the elements at the end, as `list::drop` and `list::take` do at the start.

## Merge ranges by sorting them

Puzzles with ranges, such as fresh IDs or covered columns, often need the ranges merged. Sort them by their start, then fold each range into the last merged one when they overlap or touch:

```cheby
fn merge(ranges: List<(Int, Int)>) -> List<(Int, Int)> {
  ranges
  |> list::sort
  |> list::fold(_, [], fn(merged, (low, high)) {
    case list::last(merged) {
      Some((first, last)) when low <= last + 1 => {
        list::push(list::drop_last(merged, 1), (first, int::max(last, high)))
      }
      _ => list::push(merged, (low, high))
    }
  })
}

fn covered(ranges: List<(Int, Int)>) -> Int {
  ranges
  |> merge
  |> list::map(_, fn((low, high)) { high - low + 1 })
  |> list::sum(_, 0)
}
```

The ranges include both ends. `merge([(10, 14), (3, 5), (16, 20), (12, 18)])` gives `[(3, 5), (10, 20)]`, and `covered` of the same ranges gives `14`. For ranges that leave out their end, merge when `low <= last`.

## Compress coordinates that are far apart

When a puzzle draws a few shapes with coordinates near 100,000, a grid of every cell is too large. Number the distinct values in order instead, and work on the small grid of those numbers:

```cheby
fn compress(values: List<Int>) -> map::Map<Int, Int> {
  values
  |> list::unique
  |> list::sort
  |> list::enumerate
  |> list::map(_, fn((index, value)) { (value, index) })
  |> map::from_list
}
```

`compress([100, 5, 100_000, 5])` maps `5` to `0`, `100` to `1` and `100_000` to `2`. Compress the `x` and `y` values separately. To turn a small coordinate back into a real one, keep the sorted list and read it with `list::get`. When the empty space between two values matters, as in a flood fill, add `v + 1` for each value `v` before compressing, so a gap stays a gap.

## Work with bits

`Int` has the bitwise operators `&`, `|`, `^`, `<<` and `>>` on its 64 bits, in two's complement. There is no bitwise not: `n ^ -1` flips every bit. `a << n` stops the program when the result overflows, like a multiplication, and `a >> n` keeps the sign. A shift by a negative amount or by 64 or more stops it too.

The shifts bind tighter than `&`, `&` tighter than `^`, and `^` tighter than `|`. All of them bind tighter than `|>` and the comparisons, and looser than `+` and `-`. So `mask >> bit & 1 == 1` reads as `((mask >> bit) & 1) == 1`, and `1 << n + 1` as `1 << (n + 1)`. Number literals can be written `0b1011` and `0x1f`.

`std::int` has no functions for bits, such as a count of the set bits. Write that one with a local function, which clears the lowest set bit until none is left:

```cheby
fn bit_count(mask: Int) -> Int {
  fn step(rest: Int, count: Int) -> Int {
    case rest {
      0 => count
      _ => step(rest & (rest - 1), count + 1)
    }
  }
  step(mask, 0)
}
```

`bit_count(0b1011)` gives `3`. It is for numbers from 0 up, and a negative `mask` stops it with `integer overflow`.

A mask from 0 to 2ⁿ − 1 picks a subset of n items, which makes a brute force over every subset short:

```cheby
fn subsets<T>(items: List<T>) -> List<List<T>> {
  let size = list::length(items)
  list::range(0, 1 << size)
  |> list::map(_, fn(mask) {
    items
    |> list::enumerate
    |> list::filter_map(_, fn((bit, item)) {
      case mask >> bit & 1 {
        1 => Some(item)
        _ => None
      }
    })
  })
}
```

`subsets([1, 2, 3])` gives the 8 lists from `[]` to `[1, 2, 3]`.

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

**Counting paths.** `search::count_paths(start, goal, next)` counts the paths from `start` to `goal` in a graph without cycles. It remembers the count of each state, so trillions of paths take no longer than a few. Here the graph comes from lines like `you -> a, b`, read with `rule` from [Read the input](#read-the-input-with-intfind_all-lines-and-split):

```cheby
fn routes(input: String) -> Int {
  let edges = input
    |> string::lines
    |> list::filter_map(_, rule)
    |> map::from_list
  search::count_paths("you", "out", fn(node) { map::get_or(edges, node, []) })
}
```

`routes("you -> a, b\na -> out\nb -> a, out\n")` gives `3`.

**Groups that join.** When a puzzle links items one pair at a time and asks about the groups they form, use the disjoint sets of `aoc::search`, also called union-find. `search::new_sets(keys)` starts with each key alone, `search::union(sets, a, b)` joins two groups, `search::same` and `search::find` tell which group a key is in, and `search::groups` and `search::sizes` list the groups:

```cheby
fn circuits(links: List<(Int, Int)>, boxes: Int) -> List<Int> {
  let alone = search::new_sets(list::range(0, boxes))
  let joined = list::fold(links, alone, fn(sets, (a, b)) { search::union(sets, a, b) })
  search::sizes(joined)
}
```

`circuits([(0, 1), (2, 3), (1, 3)], 6)` gives `[4, 1, 1]`: boxes 0 to 3 in one circuit, and boxes 4 and 5 alone. `sizes` lists the largest first. The sets are a value like any other, so `union` returns new sets and the old ones stay as they were.

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

Put the puzzle's example in a `test` block before you run the real input. `cheby test --module aoc::day05` then tells you whether a change still solves it. When part 2 has an example of its own, its test reads it with `kit::sample2(5)`, from `inputs/day05.sample2.txt`; see [README.md](README.md#samples).

## Habits from other languages that Cheby rejects

Each of these is a compile error, and this table saves you the round trip:

| Habit                                        | Write this instead                                                                                                  |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `let total = total + 1`                      | A new name: `let next = total + 1`. A name is bound once.                                                           |
| `fn area(region: Region)` beside `fn region` | A name of its own: `fn area(shape: Region)`. Functions, constants and imported modules are bound names too (E0044). |
| `pair.0`                                     | A pattern: `let (first, second) = pair`                                                                             |
| `xs \|> list::map(double)`                   | A hole for the piped value: `xs \|> list::map(_, double)`                                                           |
| `"{int::show(n)}"`                           | A name in the braces: `"{n}"`, after `let n = …` if it is a call                                                    |
| `"L" <> rest` as a pattern                   | `string::starts_with(line, "L")`, then `string::drop_start(line, 1)`; a pattern cannot match the start of a string  |
| `[first, .._]`                               | `[first, ..]` ignores the rest, and `[first, ..rest]` binds it                                                      |
| `list::get(xs, i).unwrap()`                  | `option::expect(list::get(xs, i), "why it is there")`, or `list::get_or(xs, i, 0)` for a default                    |
| `file::read_text(path)`                      | `io::read_file(path)`: `std::io` reads and writes whole files                                                       |
| A helper named `go`                          | `step` or `walk`; `go` is reserved                                                                                  |
| `for`, `while`, `if`, `return`               | A fold or recursion, `case`, and `guard … else { … }`                                                               |

One habit fails at run time instead: an `Int` holds 64 bits, and a result that does not fit stops the program with `integer overflow`. It never wraps around silently, so an answer that prints is not a wrapped one.
