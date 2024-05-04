# The DRY Testing Manifesto

## There is something very wrong in the land of unit tests

Unit tests, in the sense of "automated tests maintained by the owner of the code that prove it does what it's intended to do", are essential if we want to deliver software that solves real problems on schedule. The benefits — a fast feedback loop while writing code, more cases tried compared to a human manually testing after making a small fix for the Nth time, and most importantly catching bugs earlier in the development process when it's exponentially cheaper to fix them — are very much worth the cost of writing these tests.

However, the software development community has been stuck in a bad local minimum in unit test design space. The situation is so bad that it's surprising to hear about a mature project, commercial or open source, with at least one test for every feature.

It's time for a paradigm shift in how we write and maintain unit tests, a shift that will yield higher test coverage for lower time and effort, which translates to higher quality software being delivered faster and cheaper.

## The very model of a modern major unit test

For a bunch of good reasons that we will dive into later, unit tests have evolved to be written like this (example taken from [hyperformula](https://github.com/handsontable/hyperformula/blob/master/test/format/format-parser.spec.ts), a JS spreadsheet engine, chosen for the familiarity of its subject domain and the high quality of its test suite):

```javascript
describe('ParserWithCaching', () => {
  it('integer literal', () => {
    const parser = buildEmptyParserWithCaching(new Config())

    const ast = parser.parse('=42', adr('A1')).ast
    expect(ast).toEqual(buildNumberAst(42))
  })

  it('negative integer literal', () => {
    const parser = buildEmptyParserWithCaching(new Config())

    const ast = parser.parse('=-42', adr('A1')).ast as MinusUnaryOpAst
    expect(ast.type).toBe(AstNodeType.MINUS_UNARY_OP)
    expect(ast.value).toEqual(buildNumberAst(42))
  })
  
  // ...
})
```

Of course, those are just two tests out of a suite (`ParserWithCaching`) of 18 tests:

* `integer literal`
* `negative integer literal`
* `string literal`
* `plus operator on different nodes`
* `minus operator`
* `power operator`
* `power operator order`
* `joining nodes without braces`
* `joining nodes with braces`
* `float literal`
* `float literal with different decimal separator`
* `leading zeros of number literals`
* `allow to accept different lexer configs`
* `with default config should return error for non-breakable space`
* `when set ignoreWhiteSpace = 'any' should accept a non-breakable space`
* `error literal`
* `error literals are case insensitive`
* `reference to address in nonexisting range returns ref error with data input ast`

`ParserWithCaching` is one suit of 10 suites in one test file `test/parser/parser.spec.ts` out of 17 test files in the `test/parser` directory.

In total, as of today, `HyperFormula` has 4860 test files in 831 suites in 472 files. These files contain 47,093 non-blank, non-comment lines of code, for an average of ~10 lines per test (9 if you don't count the closing brackets).

## We can do better (most of the time)

Lets go back to the fundamentals. An ideal description of the first 10 tests in the `ParserWithCaching` suite would be something like:

```javascript
describe('ParserWithCaching', () => {
  parseSpreadsheet('literals', [
    ['integer', '=42'],
    ['negative integer', '=-42'],
    ['string', '="foobar"'],
    ['plus operator on different nodes', '=1+A5'],
    ['minus operator', '=1-3'],
    ['power operator', '=2^3'],
    ['power operator order', '=2*2^3'],
    ['joining nodes without braces', '=1 + 2 + 3'],
    ['joining nodes with braces', '=1 + (2 + 3)'],
    ['float literal', '=3.14'],
    // ...
  ]);
// ...
})
```

There are 10 lines of test specification (plus some off-screen scaffolding) that replace 66 lines in the original (56 not counting lines that only close brackets). Or do they? In the source there was not only "given... when...", there was also "expect", where did that go?

## The core technique: DRY Expectations

When our test runs, it already has in memory the (hopefully correct) result (in this case, the Abstract Syntax Tree of a single parsed cell). Forcing ourselves to repeat it as a literal in our test code is just creating busywork. Instead, our test could write the result to a file and then offer us the option to "bless" it, and from that point on fail the test if its result is different from the blessed result, printing their diff.

For example, for our `literals` test, our code might write a file `test/parser/gold/literals.json` like this:

```json
[
  [
    {"raw": "integer", "type": "STRING", "value": "integer"},
    {"raw": "=42", "type": "NUMBER", "value": 42}
  ],
  [
    {"raw": "negative integer", "type": "STRING", "value": "negative integer"},
    {"raw": "=-42", "type": "MINUS_UNARY_OP", "value": {"type": "NUMBER", "value": -42}}
  ],
  [
    {"raw": "string", "type": "STRING", "value": "string"},
    {"raw": "=\"foobar\"", "type": "STRING", "value": "foobar"}
  ],
  [
    {"raw": "plus operator on different nodes", "type": "STRING", "value": "plus operator on different nodes"},
    {
      "raw": "=1+A5",
      "type": "PLUS_OP",
      "left": {"type": "NUMBER", "value": 1},
      "right": {"type": "CELL_REFERENCE", "reference": "A5"}
    }
  ],
  [
    {"raw": "minus operator", "type": "STRING", "value": "minus operator"},
    {
      "raw": "=1-3",
      "type": "MINUS_OP",
      "left": {"type": "NUMBER", "value": 1},
      "right": {"type": "NUMBER", "value": 3}
    }
  ],
  [
    {"raw": "power operator", "type": "STRING", "value": "power operator"},
    {
      "raw": "=2^3",
      "type": "POWER_OP",
      "left": {"type": "NUMBER", "value": 2},
      "right": {"type": "NUMBER", "value": 3}
    }
  ],
  [
    {"raw": "power operator order", "type": "STRING", "value": "power operator order"},
    {
      "raw": "=2*2^3",
      "type": "TIMES_OP",
      "left": {"type": "NUMBER", "value": 2},
      "right": {
        "type": "POWER_OP",
        "left": {"type": "NUMBER", "value": 2},
        "right": {"type": "NUMBER", "value": 3}
      }
    }
  ],
  [
    {"raw": "joining nodes without braces", "type": "STRING", "value": "joining nodes without braces"},
    {
      "raw": "=1 + 2 + 3",
      "type": "PLUS_OP",
      "left": {
        "type": "PLUS_OP",
        "left": {"type": "NUMBER", "value": 1},
        "right": {"type": "NUMBER", "value": 2}
      },
      "right": {"type": "NUMBER", "value": 3}
    }
  ],
  [
    {"raw": "joining nodes with braces", "type": "STRING", "value": "joining nodes with braces"},
    {
      "raw": "=1 + (2 + 3)",
      "type": "PLUS_OP",
      "left": {"type": "NUMBER", "value": 1},
      "right": {
        "type": "PLUS_OP",
        "left": {"type": "NUMBER", "value": 2},
        "right": {"type": "NUMBER", "value": 3}
      }
    }
  ],
  [
    {"raw": "float literal", "type": "STRING", "value": "float literal"},
    {"raw": "=3.14", "type": "NUMBER", "value": 3.14}
  ],
  ...
]
```

The obvious disadvantage here is that the programmer might get careless and approve the wrong thing, but this usually gets caught in code review. The obvious advantage is that the effort to write a test has been reduced maybe 5x, so the developer might write 5x as many tests before losing patience and moving on. For example, they might add `=0`, `=999999999999`, `=1+2+3`, `=1-2+3`, `=1+2-3+4`, `=1+2*3`, `=1*2+3`, `=-1*2`, and many more cases that feel sorely missing when you see them listed sequentially, one per line. Another advantage is that more things are being asserted: for instance, in the original `plus operator` test, the values of the operands (`1` and `A5`) are not checked, just their types, presumably to save the poor test writer some time and sanity; here we print the complete AST, so we'll catch more mistakes "for free". In this case we could save even more typing by removing the description column, since when the cases are listed sequentially, it's very apparent from the `raw` values what each case intends to test.

This technique is known as "gold testing", "golden master testing", "snapshot testing" or "characterization testing", and there are plugins for most testing frameworks to support it, but often the best implementation is to just write to a file and then if `git status --porcelain path-to-output-file` prints any output (i.e. the file in the working tree is not identical to the committed version), call `git diff path-to-output-file` to print a colorful diff and fail the test. This has better integration with our IDE (that already has a GUI for seeing the changes in our working tree and choosing some of them to stage and commit), is easy to customize as needed to our subject domain, and is just a few lines of code to implement.

Here is example Python code to do this from a complex real-world project:

```python
import csv
from pathlib import Path
import subprocess

class Recorder:
    def __init__(self, func):
        path = Path(inspect.getfile(func)).parent
        self.path = path / "golden" / (func.__qualname__ + ".csv")
    # ...
    def write(self):
        # ...
        with self.path.open("w") as f:
            writer = csv.DictWriter(f, fieldnames=headers)
            writer.writeheader()
            writer.writerows(data)
        if subprocess.check_output(["git", "status", "--porcelain", self.path.absolute()]):
            subprocess.call(f"git diff {self.path.absolute()} | tail -n 4", shell=True)
            assert False, f"differences in file {self.path.absolute()}"
```

### DRY Expectations best practices

Treat your test code like code and Don't Repeat Yourself.

Your test case specification API should be a simple and well designed [DSL](https://en.wikipedia.org/wiki/Domain-specific_language), either embedded or in rare cases external (parsed), that allows you to specify only the crux of what you want to test, ideally in one line per case. Go all-in on [convention over configuration](https://en.wikipedia.org/wiki/Convention_over_configuration). Put thought into it and evolve it as needed.

Writing a test case should take seconds at most. Copying a case and changing one detail should be trivial (and done often, because tests should usually come in pairs that tightly hug a boundary in the space of inputs). However, if most of your cases repeat a lot of information, there should be a way to abstract that out and, in each test case, only write out the part that varies between tests.

Your output format should be very readable. Put thought into it and evolve it as needed.

It's okay to have more than one output format for different types of tests (e.g. in HyperFormula we might want one output format for parser tests and one for calculation engine tests), but try to have just a few that give you enough power to be able to test everything about the project.

Your output format should play well with diffs (so ideally a line per thing-that-might-change, but apply common sense and non-default diffing tools if your output is e.g. naturally [tabular](https://github.com/paulfitz/daff) or a [bitmap](https://github.com/mapbox/pixelmatch)).

Don't be afraid to make changes to the output format when you already have thousands of blessed tests - just make sure to run all tests before and after the output formatting changed without any changes anywhere but the testing infrastructure, and you can comfortably re-bless all the tests after a cursory glance at just a few.

Your output format should be deterministic and shouldn't contain things that change often and nobody cares: for example, you will probably want to wrap each case with a "try..catch" and print stack traces into the output file (to enable a mix of regular and `assertThrows()` tests), but you might want to remove line numbers from the stack traces or every throwing case will need to be re-blessed every time you change anything in the code, and if your language includes paths in stack traces, you'll want to remove the part above the project root if you want different team members to be able to run the same test.

If your output format can be readable to non-technical business stakeholders, that's a very nice bonus because it improves communication about requirements.

If you write nontrivial test scaffolding that doesn't obviously get exercised well by your tests, test it separately. For example, you might want to dedicate some test cases just to the output formatting being correct, but if you add a new feature to your output formatting that is only triggered by one new case you just added and doesn't break any other test, it's ok to trust your test suite to catch mistakes in the new formatting code.

## Further techniques

There are many patterns that appear often when you approach writing tests this way, and they will be covered more extensively later.

### VCR

If you have an external dependency that exposes an API (e.g. REST or a DB), you might want to use the [VCR](https://github.com/vcr/vcr) pattern to be able to run your tests either in "record" (integration test) mode or in "playback" (unit test, mocked dependencies) mode.

Don't get confused - you will have two kinds of recordings in your git repo: the requests and responses made by the test, as well as the blessed test output. Both are needed, they play different roles and should be stored separately.

## A paradigm shift, not a hype cycle

For a successful paradigm shift, first we must understand the existing paradigm and what it gets right, otherwise you just get a hype cycle[^1]. So here is a quick overview of unit testing methodology, and if any of this isn't familar, consider reading e.g. the testing chapters from [this](https://abseil.io/resources/swe-book/html/toc.html).

### Why we write unit tests the way we do

These are some things we'd want from a unit test:

* **Run really fast, ideally in milliseconds**: a shorter feedback loop between writing code and knowing it has an issue speeds up development exponentially
* **Test one bit of behavior**: in a 100kLOC codebase, "you introduced a bug with parsing empty strings" saves us a lot of time debugging in production but just "you introduced a bug" wastes way more time than it saves relative to only debugging bugs that users complain about
* **Test the external behavior and not internal implementation details**: we want to be able to refactor our code in the sense of "improving the implementation without breaking the tests", and we want to add a feature without fixing all existing tests, otherwise development will be *O(n^2)* and the project will grind to a halt
* **Tests order shouldn't matter**: else when you have thousands of tests and need to change one of them, or want to enable parallelization to keep their runtime reasonable, the project will grind to a halt
* **Be reproducible, i.e. deterministic**: otherwise tests failing don't imply that your latest change broke something, only that some change introduced in the last few weeks broke something, which wastes a lot of time by assigning someone unfamiliar with a change to hunt for an issue caused by that change
* **Be obvious and readable**: we don't test our tests, so even simple bugs in the test logic have to be found the hard way

Lets look again at our model unit test:

```javascript
describe('ParserWithCaching', () => {
  beforeEach(() => {
    unregisterAllLanguages()
    HyperFormula.registerLanguage(plPL.langCode, plPL)
    HyperFormula.registerLanguage(enGB.langCode, enGB)
  })

  it('integer literal', () => {
    const parser = buildEmptyParserWithCaching(new Config())

    const ast = parser.parse('=42', adr('A1')).ast
    expect(ast).toEqual(buildNumberAst(42))
  })
// ...
})
```

There is setup which involves building all the objects that play a part in the scenario (some of which might be mocks to remove non-determinism or slow dependencies, and in one variant also to prevent tests being failed by changes to other units of code), then a single method call, then a comparison of the result of that method call with what amounts to a predefined constant value. Some call this pattern "given... when... expect". There are thousands of tests like this, each with one line of "when" (some say that "unit tests should be DAMP, not DRY" i.e. instead of practicing the Don't Repeat Yourself principle you should try to write code that gives Descriptive And Meaningful Phrases, each phrase being standalone).

How does this achieve our requirements?

* **Fast** fakes/mocks
* **One bit of behavior** DAMP not DRY
* **External behavior** we assert on the return value, not internal state
* **Order Invariance** per-test setup/teardown
* **Reproducible** if we had randomness we would seed it with 0, if external dependencies are involved we would mock them
* **Obvious** no state saved between tests, single line of "when", compare to literal values in "expect"


[^1] the ruling idea "X is the only way" is replaced by the "X is literally the worst", people do the exact opposite of X for a while until it finally becomes socially acceptable to say "ok but we are now even worse, maybe X had good parts too, there's a reason it ruled", which paves the way for the new idea "the exact opposite of X is literally the worst" to rise to power for a few years, rinse and repeat

