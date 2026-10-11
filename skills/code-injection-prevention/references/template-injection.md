# Template injection (SSTI)

An attacker supplies input that the server renders as template source
rather than template data. The template engine evaluates the attacker's
code with the server's full privileges — reading environment variables,
calling methods on server-side objects, importing modules, and executing
arbitrary commands. The fix is structural: user input is data
(variables), never source (template code). If the template string itself
never contains attacker-controlled text, injection is impossible.

The principle is language-agnostic: every template engine (Jinja2,
Handlebars, ERB, Twig, Thymeleaf, Go `html/template`, Pug, etc.)
distinguishes template source from template data. The examples below use
Python with Jinja2; the same pattern applies to every engine.

## ❌ Before (vulnerable)

```python
from jinja2 import Environment

env = Environment()

# UNSAFE — user input is compiled as template source.
# An attacker can supply: user_input = "{{ self.__init__.__globals__ }}"
# or worse: "{{ ''.__class__.__mro__[1].__subclasses__() }}"
# to inspect server internals and escalate to RCE.

def render_greeting(user_input):
    template = env.from_string(f"Hello, {user_input}!")
    return template.render()
```

`env.from_string(f"Hello, {user_input}!")` hands the template engine a
string containing attacker-controlled text. The engine parses it for
`{{ }}` expressions, `{% %}` blocks, and filters — the attacker
controls all of them.

## ✅ After (safe)

```python
from jinja2 import Environment

env = Environment()

# SAFE — user input is passed as a template variable.
# The template string is static authored code; the user's value
# is interpolated as data, not parsed as template syntax.

GREETING_TEMPLATE = env.from_string("Hello, {{ name }}!")

def render_greeting(user_input):
    return GREETING_TEMPLATE.render(name=user_input)
```

The template string `"Hello, {{ name }}!"` is authored code — static
and controlled. `user_input` flows in as the value of `name` and is
never parsed for template syntax.

## Key points

- **Template source is authored code.** Treat it like any other source
  file — it is written by developers, stored in the repository, and
  never constructed from external input.
- **Compile templates at startup or deploy time.** Loading templates
  from a static directory or embedding them as constants makes it
  structurally impossible for user input to become template source.
- **Sandboxed template engines reduce blast radius but do not fix the
  bug.** Jinja2's `SandboxedEnvironment` restricts attribute access,
  but sandbox escapes are found regularly — the fix is not rendering
  user input as template source in the first place.
- **Autoescape is orthogonal.** Jinja2's `autoescape=True` and
  equivalent settings prevent XSS in the rendered output — they do not
  prevent SSTI. An autoescaped template that compiles user input as
  source is still vulnerable to server-side code execution.
- **Watch for indirect template construction.** Building a template from
  a database field, a config value fetched from an API, or a file
  uploaded by a user all introduce the same risk — if the source is not
  developer-authored and immutable, it is user input.
