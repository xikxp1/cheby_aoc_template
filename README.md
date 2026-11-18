# Advent of Code in Cheby

This template saves each [Advent of Code](https://adventofcode.com) day in its own Cheby module. It downloads your input, times both parts, and submits answers:

```sh
cheby run -- new 5                  # start day 5: write src/day05.cheby, download input and sample
cheby run aoc::day05 -- --sample    # both parts on the puzzle's example
cheby run aoc::day05                # both parts on your input, with timings
cheby run aoc::day05 -- --submit 1  # send part 1's answer and print the site's reply
```

Run every command from this directory, because paths such as `inputs/day05.txt` are relative to it.

## Set up once

1. **Install `cheby`.** It must be new enough to have `option::expect`, `map::get_or` and `list::drop_last`, which the template's modules use.
2. **Give the template your session cookie.** Log in to adventofcode.com in a browser and copy the value of the `session` cookie. Put it in the environment as `AOC_SESSION`, or in a file named `.session` in this directory. Git ignores `.session`.
3. **Edit `src/config.cheby`.** Set `year`, and `days` to the number of puzzle days: 12 since 2025, 25 before. In `user_agent`, replace `<your contact>` with a way to reach you, such as your email or the URL of the repository with your solutions. Advent of Code asks automated tools to say who sends their requests, so it can reach that person if a script misbehaves.

Each input is downloaded once and kept in `inputs/`. Git ignores that directory, because Advent of Code asks people not to publish their inputs.

## Solve a day

`cheby run -- new 5` writes `src/day05.cheby`:

```cheby
//! Advent of Code 2026, day 5.

import aoc::kit

/// Part 1.
fn part1(input: String) -> Int {
  todo as "day 5, part 1"
}

/// Part 2.
fn part2(input: String) -> Int {
  todo as "day 5, part 2"
}

fn main() -> Result<Nil, kit::Error> {
  kit::run(5, part1, part2)
}

// `cheby test --module aoc::day05` checks the parts on the samples:
// `inputs/day05.sample.txt`, and for part 2 `inputs/day05.sample2.txt`
// when it exists. Put the puzzle's answers for the samples here.

test "part 1 on the sample" {
  let assert Ok(sample) = kit::sample(5)
  assert part1(sample) == 0
}

test "part 2 on the sample" {
  let assert Ok(sample) = kit::sample2(5)
  assert part2(sample) == 0
}
```

It also saves the first code block of the puzzle's page as `inputs/day05.sample.txt`. That block is usually the example, but not always, so check the file before you trust it.

Then work through the parts:

1. Replace the `todo` in `part1` with your solution, and replace the `0` in its test with the example's answer from the puzzle. Until then, running the day stops at the `todo` with `not implemented: day 5, part 1`.
2. Run `cheby test --module aoc::day05` until the test passes.
3. Run `cheby run aoc::day05 -- --part 1` to solve your input, then `cheby run aoc::day05 -- --submit 1` to send the answer.
4. Do the same for part 2.

A part can return any type that has `show`, such as `Int` or `String`. The input reaches it unchanged, final line break included, so trim it when a part needs to.

If the puzzle is not open yet, `new` still writes the module and reports that the download failed. It then skips the puzzle's page, so a closed day costs one request. Run `cheby run -- fetch 5` once the puzzle opens.

### Samples

Some puzzles give part 2 an example of its own. Once you solve part 1, run `cheby run -- fetch 5` again: it saves the first code block of part 2 as `inputs/day05.sample2.txt`. If part 2 has no code block, or its first one is not the example, write that file by hand, or delete it.

`--sample` runs part 2 on `inputs/day05.sample2.txt` when the file exists, and on `inputs/day05.sample.txt` otherwise. `kit::sample2(5)` reads the same file for part 2's test.

### When the sample needs other numbers, or the parts share their parsing

`kit::run` passes each part only the input. Two other forms of `main` give the parts more:

- **`kit::run_with(5, part1, part2)`** passes each part the input and a `Bool` that is `True` on `--sample`. Use it when the sample asks for other numbers than your input, such as 10 steps instead of 1000. The parts are `fn part1(input: String, on_sample: Bool) -> Int`.
- **`kit::run_parsed(5, parse, part1, part2)`** runs `parse(input, on_sample)` once, prints its time on a line of its own, and passes its result to both parts. The parts are `fn part1(puzzle: Puzzle) -> Int`, for the type that `parse` returns. When part 2 has its own sample, `--sample` parses each sample once.

`--sample`, `--input`, `--part` and `--submit` work the same with all three forms. [GUIDE.md](GUIDE.md#parse-once-for-both-parts-with-kitrun_parsed) shows an example.

### The last day

The last day of the year, `days` in `src/config.cheby`, has only part 1. `new` writes a skeleton for it without `part2`, whose `main` is `kit::run_one(12, part1)`. It runs and tests part 1 only.

### Remove a day

`cheby run -- remove 5` deletes `src/day05.cheby` and rewrites `src/all.cheby` without it. The inputs stay in `inputs/`. Delete a day this way, not by hand: `src/all.cheby` imports every day, so a day deleted by hand stops every `cheby run`, `sync` included.

### If `all.cheby` breaks

If every `cheby run` fails in `src/all.cheby`, for example because a day's file was deleted by hand, empty that file's list and its imports, then let `sync` write them again:

```sh
printf 'import aoc::kit\n\nfn main() -> Result<Nil, kit::Error> {\n  kit::run_all([])\n}\n' > src/all.cheby && cheby run -- sync
```

## Commands and flags

| Command                                       | What it does                                                                                      |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `cheby run -- new N`                          | Writes `src/dayNN.cheby` unless it exists, then runs `sync` and `fetch N`. N is from 1 to `days`. |
| `cheby run -- fetch N`                        | Downloads day N's input, then its samples. Files that exist are kept.                             |
| `cheby run -- remove N`                       | Deletes `src/dayNN.cheby` and runs `sync`. The inputs are kept.                                   |
| `cheby run -- sync`                           | Rewrites `src/all.cheby` from the `src/dayNN.cheby` files.                                        |
| `cheby run aoc::dayNN`                        | Runs both parts on your input.                                                                    |
| `cheby run aoc::dayNN -- --sample` (`-s`)     | Runs both parts on the samples, part 2 on `dayNN.sample2.txt` if it exists.                       |
| `cheby run aoc::dayNN -- --input FILE` (`-i`) | Runs both parts on `FILE`.                                                                        |
| `cheby run aoc::dayNN -- --part 2` (`-p`)     | Runs one part only. Combines with `--sample` and `--input`.                                       |
| `cheby run aoc::dayNN -- --submit 2`          | Solves part 2 on your input and sends the answer. Refuses `--sample` and `--input`.               |
| `cheby run aoc::all`                          | Runs every day on its input and prints the total time.                                            |
| `cheby test --module aoc::dayNN`              | Runs the day's `test` blocks.                                                                     |

## Helpers

The template has three modules of its own. Import them like any other module, for example `import aoc::grid`.

- **`aoc::kit`** runs a day, reads and downloads inputs, and submits answers. Each day's `main` calls `run`, `run_with`, `run_parsed` or `run_one`, and tests read the samples with `sample` and `sample2`.
- **`aoc::grid`** reads a character map into a `Grid`. A position is `(x, y)`, with `y` counting rows downward from the top. It has `parse`, `get`, `set`, `in_bounds`, `positions`, `find`, `find_all` and `draw`, the directions `directions4` and `directions8`, and the moves `add`, `turn_right`, `turn_left`, `neighbours4`, `neighbours8` and `manhattan`.
- **`aoc::search`** has `bfs` and `bfs_to` for step counts, `dijkstra` and `dijkstra_all` for weighted paths, `count_paths` for the number of paths in a graph without cycles, and `find_cycle` and `state_after` for puzzles that ask about a billion steps. Its disjoint sets, or union-find, track groups that join: `new_sets`, `union`, `find`, `same`, `groups` and `sizes`.

The standard library also has helpers that puzzles need often:

| Helper                                        | Example                                                                   |
| --------------------------------------------- | ------------------------------------------------------------------------- |
| `int::find_all(text)`                         | `"p=3,-4 v=12"` gives `[3, -4, 12]`, and the range `"2-4"` gives `[2, 4]` |
| `int::gcd`, `int::lcm`                        | `list::fold(cycle_lengths, 1, int::lcm)`                                  |
| `list::windows`, `list::chunks`               | `list::windows([1, 2, 3], 2)` gives `[[1, 2], [2, 3]]`                    |
| `list::transpose`                             | `[[1, 2], [3, 4]]` gives `[[1, 3], [2, 4]]`, columns as rows              |
| `list::pairs`, `combinations`, `permutations` | `list::pairs([1, 2, 3])` gives `[(1, 2), (1, 3), (2, 3)]`                 |
| `map::frequencies(xs)`                        | `["a", "b", "a"]` gives a map with `"a"` at 2 and `"b"` at 1              |
| `function::iterate(x, n, f)`                  | `f` applied `n` times: 100 rounds of a simulation                         |
| `function::fixpoint(x, f)`                    | `f` applied until the value stops changing                                |
| `heap::new_min()`, `push`, `pop`              | a priority queue that keeps equal elements                                |
| `io::debug(x)`                                | prints `x` to standard error and returns it, so it fits inside a pipe     |

## Writing solutions in Cheby

[GUIDE.md](GUIDE.md) shows the patterns that puzzles need most often, each with code that runs: reading the input, counting groups, caching results by hand, merging ranges, working with bits, and searching. In short:

- **Repeat with recursion.** Cheby has no loops. Use `list::fold`, `function::iterate`, or a local recursive `fn`.
- **Recurse deeply if you need to.** `cheby.toml` gives `main` a 1 GiB stack, so a flood fill over a large grid does not overflow it.
- **Key grids by tuple.** `Map<(Int, Int), T>` and `Set<(Int, Int)>` work well for grids and visited sets.
- **Index lists freely.** A `List` is a vector, not a linked list: `list::get` and `list::set` take O(log n).
