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

1. **Install `cheby`.** It must be new enough to have `int::find_all`, `std::heap` and `std::function`, which the helpers use.
2. **Give the template your session cookie.** Log in to adventofcode.com in a browser and copy the value of the `session` cookie. Put it in the environment as `AOC_SESSION`, or in a file named `.session` in this directory. Git ignores `.session`.
3. **Edit `src/config.cheby`.** Set `year`. In `user_agent`, replace `<your contact>` with a way to reach you, such as your email or the URL of the repository with your solutions. Advent of Code asks automated tools to say who sends their requests, so it can reach that person if a script misbehaves.

Each input is downloaded once and kept in `inputs/`. Git ignores that directory, because Advent of Code asks people not to publish their inputs.

## Solve a day

`cheby run -- new 5` writes `src/day05.cheby`:

```cheby
fn part1(input: String) -> Int {
  todo as "day 5, part 1"
}

fn part2(input: String) -> Int {
  todo as "day 5, part 2"
}

fn main() -> Result<Nil, kit::Error> {
  kit::run(5, part1, part2)
}

test "part 1 on the sample" {
  let assert Ok(sample) = kit::sample(5)
  assert part1(sample) == 0
}
```

It also saves the first code block of the puzzle's page as `inputs/day05.sample.txt`. That block is usually the example, but not always, so check the file before you trust it.

Then work through the parts:

1. Replace the `todo` in `part1` with your solution, and replace the `0` in its test with the example's answer from the puzzle. Until then, running the day stops at the `todo` with `not implemented: day 5, part 1`.
2. Run `cheby test --module aoc::day05` until the test passes.
3. Run `cheby run aoc::day05 -- --part 1` to solve your input, then `cheby run aoc::day05 -- --submit 1` to send the answer.
4. Do the same for part 2.

A part can return any type that has `show`, such as `Int` or `String`. The input reaches it unchanged, final line break included, so trim it when a part needs to.

If the puzzle is not open yet, `new` still writes the module and reports that the download failed. Run `cheby run -- fetch 5` once the puzzle opens.

## Commands and flags

| Command                                       | What it does                                                               |
| --------------------------------------------- | -------------------------------------------------------------------------- |
| `cheby run -- new N`                          | Writes `src/dayNN.cheby` unless it exists, then runs `sync` and `fetch N`. |
| `cheby run -- fetch N`                        | Downloads day N's input and sample. Files that exist are kept.             |
| `cheby run -- sync`                           | Rewrites `src/all.cheby` from the `src/dayNN.cheby` files.                 |
| `cheby run aoc::dayNN`                        | Runs both parts on your input.                                             |
| `cheby run aoc::dayNN -- --sample` (`-s`)     | Runs both parts on `inputs/dayNN.sample.txt`.                              |
| `cheby run aoc::dayNN -- --input FILE` (`-i`) | Runs both parts on `FILE`.                                                 |
| `cheby run aoc::dayNN -- --part 2` (`-p`)     | Runs one part only. Combines with `--sample` and `--input`.                |
| `cheby run aoc::dayNN -- --submit 2`          | Solves part 2 on your input and sends the answer.                          |
| `cheby run aoc::all`                          | Runs every day on its input and prints the total time.                     |
| `cheby test --module aoc::dayNN`              | Runs the day's `test` blocks.                                              |

## Helpers

The template has three modules of its own. Import them like any other module, for example `import aoc::grid`.

- **`aoc::kit`** runs a day, reads and downloads inputs, and submits answers. Each day's `main` calls it.
- **`aoc::grid`** reads a character map into a `Grid`. A position is `(x, y)`, with `y` counting rows downward from the top. It has `parse`, `get`, `set`, `in_bounds`, `positions`, `find`, `find_all` and `draw`, the directions `directions4` and `directions8`, and the moves `add`, `turn_right`, `turn_left`, `neighbours4`, `neighbours8` and `manhattan`.
- **`aoc::search`** has `bfs` and `bfs_to` for step counts, `dijkstra` and `dijkstra_all` for weighted paths, and `find_cycle` and `state_after` for puzzles that ask about a billion steps.

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

[GUIDE.md](GUIDE.md) shows the patterns that puzzles need most often, each with code that runs: reading the input, counting groups, caching results by hand, and searching. In short:

- **Repeat with recursion.** Cheby has no loops. Use `list::fold`, `function::iterate`, or a local recursive `fn`.
- **Recurse deeply if you need to.** `cheby.toml` gives `main` a 1 GiB stack, so a flood fill over a large grid does not overflow it.
- **Key grids by tuple.** `Map<(Int, Int), T>` and `Set<(Int, Int)>` work well for grids and visited sets.
