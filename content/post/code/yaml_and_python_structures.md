---
title: "Loading non-primitives with PyYAML, safely!"
date: 2023-04-21T16:09:54+03:00
---

>**TL;DR:** PyYAML doesn't translate Python non-primitive data types, but using unsafe loads might result in a deserialization attack.

YAML is a very useful markup language for representing data in a human-readable format.

It is very much like JSON in that part (except for obvious formatting difference), only that it is a superset of JSON.
Meaning, it contains features included in JSON, and more.
One such feature is the ability to tag the type of data in the file as **custom data types**.

For me it was useful as I was working on converting a very complicated Python object into YAML, which had tuples.

## My tupley YAML

Here's an example of the YAML I needed to work with:

```yaml
correlation:
- !!python/tuple
  - 35000
  - .inf
  - 3.3
```

This should translated to the following Python object:

```python
{ 'correlation': 
    [
    (35000, float('inf'), 3.3)
    ]
}
```

BUT...
When I tried using PyYAML's loader to translate the yaml string into a Python dictionary:
```python
yaml.load(yaml_string, Loader=yaml.Loader)
import yaml
```

I opened a unintended breach for remote code execution! 😱

## is `yaml.load`... an unsafe load?!

Yes, apparently. 😬

Even PyYAML's official documentation states that `yaml.load` isn't safe (see "Loading YAML" under [these docs](https://pyyaml.org/wiki/PyYAMLDocumentation)).

The reason for this unsafety is that PyYAML will accept **any** custom type.

It will include `!!python/tuple` which is what we want, but also `!!python/object/apply:subprocess.Popen`, which opens a new subprocess in Python. This basically allows code execution like `os.system()`.

You can read more about it [in this article on pentesting](https://book.hacktricks.xyz/pentesting-web/deserialization/python-yaml-deserialization) and [a lengthier paper about deserialization attacks in Python](https://www.exploit-db.com/docs/english/47655-yaml-deserialization-attack-in-python.pdf).

## An easy mitigation

The best solution would be to use:

```python
yaml.safe_load()
```

or

```python
yaml.load(Loader=yaml.SafeLoader)
```

And add an explicit constructor for the Python types you want to add, in this case a tuple:

```python
def python_tuple_constructor(loader, node):
    value = loader.construct_sequence(node)
    return tuple(value)

yaml.SafeLoader.add_constructor('tag:yaml.org,2002:python/tuple', python_tuple_constructor)
```