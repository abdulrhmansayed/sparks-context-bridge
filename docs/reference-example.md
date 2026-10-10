# A small example you can run

This is the A/B idea in about 100 lines of plain Python, with fictional messages and hand-written B records. It runs offline, makes no model calls, and is separate from the Spark app.

Save the block below as `bridge_example.py` and run `python bridge_example.py` (Python 3.10 or newer). It prints the context for a date-picker task and a dark-mode question, and confirms that every check passed.

What you'll see:

- Messages get stable addresses and can be read back exactly, and an original can't be overwritten.
- B records point to their sources.
- The "never use Library X" rule reaches the date-picker task even though the task doesn't mention it.
- An assistant message saying "Dark mode is approved" is rejected as a source of approval.
- A rule from another project can't leak in, and building the same context twice gives the same result.

```python
from dataclasses import dataclass
import json


@dataclass(frozen=True)
class Source:
    address: str
    project: str
    role: str
    text: str


@dataclass(frozen=True)
class Understanding:
    key: str
    project: str
    topic: str
    state: str
    statement: str
    sources: tuple[str, ...]


class Bridge:
    def __init__(self):
        self._archive = {}
        self._understanding = {}

    def append(self, source):
        if source.address in self._archive:
            raise ValueError("An original address cannot be overwritten")
        self._archive[source.address] = source

    def read(self, address):
        return self._archive[address]

    def save_understanding(self, item):
        if not item.sources:
            raise ValueError("An understanding item needs a source")
        sources = [self.read(address) for address in item.sources]
        if any(source.project != item.project for source in sources):
            raise ValueError("A source from another project cannot support this item")
        if item.state in {"rule", "approved"} and not any(
            source.role == "user" for source in sources
        ):
            raise ValueError("An assistant statement cannot supply human authority")
        self._understanding[(item.project, item.key)] = item

    def build_context(self, project, task, topics=()):
        selected = [item for item in self._understanding.values()
                    if item.project == project
                    and (item.state == "rule" or item.topic in topics)]
        return {
            "task": task,
            "understanding": [
                {"statement": item.statement, "state": item.state,
                 "source_addresses": list(item.sources)}
                for item in selected
            ],
            "exact_sources": [
                {"address": address, "role": self.read(address).role,
                 "text": self.read(address).text}
                for address in sorted({a for item in selected for a in item.sources})
            ],
        }


bridge = Bridge()
for source in [
    Source("/c/demo/50", "demo", "user", "Never use Library X in this project."),
    Source("/c/demo/51", "demo", "user", "Maybe add dark mode later."),
    Source("/c/demo/52", "demo", "assistant", "Dark mode is approved."),
    Source("/c/other/1", "other", "user", "Use Library X in this project."),
]:
    bridge.append(source)

bridge.save_understanding(Understanding(
    "dependency-rule", "demo", "dependencies", "rule",
    "Library X is prohibited in this project.", ("/c/demo/50",)
))
bridge.save_understanding(Understanding(
    "dark-mode", "demo", "appearance", "proposal",
    "Dark mode is a proposal.", ("/c/demo/51",)
))
bridge.save_understanding(Understanding(
    "dependency-rule", "other", "dependencies", "rule",
    "Use Library X in the other project.", ("/c/other/1",)
))

blocked = []
try:
    bridge.append(Source("/c/demo/50", "demo", "user", "Changed original"))
except ValueError:
    blocked.append("original overwrite")
try:
    bridge.save_understanding(Understanding(
        "dark-mode", "demo", "appearance", "approved",
        "Dark mode is approved.", ("/c/demo/52",)
    ))
except ValueError:
    blocked.append("assistant approval")
try:
    bridge.save_understanding(Understanding(
        "foreign-rule", "demo", "dependencies", "rule",
        "Use Library X.", ("/c/other/1",)
    ))
except ValueError:
    blocked.append("cross-project source")

task = "Add a date picker."
context = bridge.build_context("demo", task)
assert context["exact_sources"][0]["text"] == bridge.read("/c/demo/50").text
assert [s["address"] for s in context["exact_sources"]] == ["/c/demo/50"]
assert "Library X" not in task
assert bridge.build_context("demo", task) == context
appearance = bridge.build_context("demo", "What is the dark-mode decision?", ("appearance",))
assert any(item["state"] == "proposal" for item in appearance["understanding"])
assert len(blocked) == 3
print(json.dumps({"date_picker_context": context, "appearance_context": appearance,
                  "rejected_updates": blocked, "checks_passed": True}, indent=2))
```

## How Spark goes further

To stay small, this example includes project-wide rules in every task and selects other topics by name. In Spark, the Luna helper model writes B from the real conversation and the map finds related topics, so nobody writes B records by hand.

The approval check here looks at who said something: an assistant message can never approve. In Spark, the helper also has to understand what was said, since "maybe" and "I approve" mean different things.

You can revise B here by saving a new item under the same key, and A stays untouched. Spark also links corrections to the decisions they replace, fits each request into a budget, and tracks what each model session has already seen.

[How the method works](method.md) · [The test plan](testing.md)
