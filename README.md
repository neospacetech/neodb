# NeoDB Core

NeoDB is an embeddable, schema-aware database engine built around NeoQL: one
query language for tables, graphs, documents, key/value data, and vectors.

> Status: early Python MVP. Dataset creation, record insertion, filtering,
> ordering, projection, and pagination are the first implementation target.

## Quick start

NeoDB supports Python 3.10 through 3.14 and has no third-party runtime
dependencies.

```bash
git clone https://github.com/neospacetech/neodb.git
cd neodb/engine
python -m pip install -e .
neodb
```

Execute a NeoQL script with `neodb path/to/script.neoql`. The shell shows
`... ` while delimiters or string literals remain open. Scripts may contain
comments, blank lines, top-level semicolon-separated statements, or one
complete statement per line. Multiline statements remain buffered until
complete.

Script execution returns `0` on success, `1` for a NeoQL diagnostic, and `2`
when the source file cannot be read. Diagnostics include the source filename
and global line and column.

Example session:

```neoql
create dataset users(table{id(int, pk), name(str(255)), age(int)})
add {id=1, name="Alice", age=25}, {id=2, name="Ben", age=17} into users
users({age>=18}).(name, age).order(age desc).limit(20)
```

Run the tests with:

```bash
cd engine && python -m unittest discover -v
```

## Development

Install the development toolchain and run all local quality gates:

```bash
cd engine
python -m pip install -e ".[dev]"
ruff format --check .
ruff check .
mypy cli datasets neoql scripts engine.py storage.py
coverage run -m unittest discover -v
coverage report
python -m build
```

Coverage is enforced at 70%. Continuous integration runs the suite on every
supported Python version and publishes coverage XML plus built distributions as
workflow artifacts. See [CONTRIBUTING.md](docs/CONTRIBUTING.md) for the contribution
workflow, [RELEASING.md](docs/RELEASING.md) for trusted releases and TestPyPI dry
runs, and the
[NeoDB Core Roadmap](https://github.com/orgs/neospacetech/projects/3) for current
progress.

## Language frontend

NeoQL source is tokenized and parsed independently of the CLI:

```python
from neoql import parse_statement

statement = parse_statement("users({age>=18}).(name, age).limit(20)")
```

The returned typed AST is immutable and every node carries a source span with
line and column positions. Syntax errors use those spans to render a precise
diagnostic and source caret. The current engine adapter converts supported AST
statements into the MVP execution contract while the lazy planner is developed.

Parsed schemas pass through NeoQL's semantic type system before reaching the
engine. The public type API supports validation, display, serialization,
literal inference, and explicit casting:

```python
from neoql import cast_value, infer_type, parse_type

user_id = parse_type("uuid")
tags = parse_type("list(str(32))")
inferred = infer_type([1, 2, 3])
identifier = cast_value("12345678-1234-5678-1234-567812345678", user_id)
```

## Schema enforcement

Table schemas are runtime contracts. Fields are required unless they are
nullable or have a default, and values are cast through their declared NeoQL
type before storage:

```neoql
create dataset users(
    table{
        tenant_id(int, pk),
        id(int, pk),
        email(str(255), unique, index),
        display_name(str(80), default("Anonymous")),
        nickname(str(80), nullable),
        biography(text, searchable),
        embedding(list(float), vector),
        created_by(str(80), readonly)
    }
)
```

Primary keys and unique values are checked across existing records and an
entire incoming batch before anything is committed. Updates use the same type,
nullability, uniqueness, and readonly rules. The schema also exposes index,
vector, and search metadata for later planners.

Constraint failures raise `ConstraintViolation`, whose `to_dict()` result
contains a stable error category, code, dataset, field, message, offending
value when applicable, and additional conflict details.

## Roadmap

- Complete the NeoQL parser and typed abstract syntax tree
- Enforce schemas and constraints
- Add document, vector, and relationship storage
- Build lazy selections and an execution planner
- Add persistence, write-ahead logging, indexes, and ACID transactions
- Ship SDKs and remote HTTP/WebSocket access

---

# NeoQL specification

The canonical NeoQL language specification (Draft v0.1) lives in [neospacetech/neoql](https://github.com/neospacetech/neoql/blob/main/spec/neoql.md).
This engine conforms to it; see `engine/tests/test_spec_conformance.py`.
