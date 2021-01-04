# gitlab-groomer

Gitlab Groomer is inspired by gitlab-triage bot. It is a simple CLI executable to triage Gitlab Epics.

## Installation

```
pip install gitlab-groomer
```

or if your Python 3.x is using `pip3`, then use:

```
pip3 install gitlab-groomer
```

## Using Gitlab-Groomer

Set the following environmental variables:

* GITLAB_URL: the URL of the Gitlab instance to use. For example: `https://gitlab.local`
* PRIVATE_TOKEN: use this private token. For example: FJKS934FSKW234
* GROUP_ID: the Gitlab group id to apply the rules to.
* RULES: directory containing the rules

To execute, simply run:

```
gitlab-groomer apply
```

The definitions directory should contain .yaml files.
Gitlab Groomer will pick up any files ending with `.yaml` in that directory.

### Structure of rule files

The groomer applies `rules` against *open Epics* in various Gitlab groups. A rule is identified by a `name`, which can be any arbitrary string.

Rules can specify specific group ids to which they apply. Otherwise rules are applied to the group specified in the `GROUP_ID` environmental variable.

In addition to `name`, a rule consists of `spec`, and zero of more `examples`.

### rule spec

There are two ways to specify rule spec.

* Option 1 - direct transformation

In this way, rule spec is a `jq` transformation, which takes as an input a (augmented) Epic record, and transforms it into an array of `actions`.

* Option 2 - seperate conditions and actions

In this way, the rule spec is on object with two keys: `conditions` and `actions`. Each `condition` and `action` is a `jq` transformation. Conditions are expected to result in a `true` or `false` value, and actions are expected to result in an `action` object. See [examples](examples/).

### Types of actions

* update epic

```
- type: update_epic
  payload:
    description: lorem posem
    add_labels: label1,label2

```

The payload is going to be sent to Gitlab [update epic API](https://docs.gitlab.com/ee/api/epics.html#update-epic). If the `id` and `epic_iid` fields are not specified, it is assumed to be the epic that matched the rule is applied to.

* create epic note

```
- type: create_note
  payload:
    body: this is a note
```

The payload is going to be sent to Gitlab [create new epic note API](https://docs.gitlab.com/ee/api/notes.html#create-new-epic-note). If the `id` and `epic_iid` fields are not specified, it is assumed to be the epic that matched the rule is applied to.

* create new issue

```
- type: create_issue
  payload:
    id: 555
    labels: label5,label7
```

The payload is going to be sent to Gitlab [new issue API](https://docs.gitlab.com/ee/api/issues.html#new-issue). If the `epic` field is not specified, it is assumed to be the epic that matched the rule is applied to.

### example of transformation

The following example will add a `needs attention` label to any epic that does not have a label that starts with `type`.

```
call attention to tickets without type:
  spec: |
    if ( .labels | .[] | test("type::.*")) then
    [
      {
        type: "update_epic",
	payload: {
	  add_labels: "needs attention"
	}
      }
    ]
    else null
    end
```

The following example will add an issue with label `breakdown task` to project 444 for any epic that doesn't have one.
```
add breakdown task:
  spec: |
    if (.issues | .[] | .labels | .[] | test("breakdown task")) then
    [ { type: "create_issue", payload: {id: 444, title: "breakdown epic " + .title, labels: "breakdown task"}}] else null end
```

### examples

A rule can have one of more examples. Examples are static epic records, and the list of actions that the rule is expected to transform them into. This is useful for regression testing as well as for debugging rules.

In the second example below:

```
add breakdown task:
  spec: |
    if (.issues | .[] | .labels | .[] | test("breakdown task")) then
    [ { type: "create_issue", payload: {id: 444, title: "breakdown epic " + .title, labels: "breakdown task"}}] else null end
  examples:
  - input:
      title: example epic
      issues: []
    expected:
    - type: create_issue
      payload:
        id: 444
	title: breakdown epic example epic
	labels: breakdown task
```

### Using Examples

#### Testing an example

To test that the rule returns the expected action:

```
gitlab-groomer test
```

To test the example below, assuming the rule is saved in `./rules/rules.yaml`:

```
gitlab-groomer --rule "add breakdown task" test
```

#### Generating an example

To generate an example, run:

```
gitlab-groomer [--rule RULE_NAME] generate-example [--epic-iid EPIC_IID]
```
