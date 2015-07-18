## Warning
**This document is intended only to serve as a guide during the design phase.  Once the design stabilizes, it is likely to diverge from reality, and be marked as deprecated or defunct; it may get deleted as well!**

## Database
`flow` maintains its state in an RDBMS of user's choice, as long as a `database/sql`-compliant driver is available for it.  [_Currently, MySQL, PostgreSQL and SQLite3 appear to be best supported._]  Development is being done against MySQL currently.  Out-of-the-box support for more databases may arrive in time.

### Entities
The core entity types in the system are:

- User,
- Role,
- Group,
- Privilege,
- Resource,
- Document,
- DocType,
- DocEvent,
- DocState,
- Workflow,
- FlowNode,
- NodeType,
- Mailbox and
- Message.

Each of the above is described hereunder, using an initial Go representation of the data held in it.

#### User
User details are expected to be provided by an external identity provider application or directory.  `flow` neither defines nor manages users.

For its own purposes, though, `flow` keeps track of the following details of each user entering the system.

```go
type User struct {
    id     uint64 // must be globally-unique
    name   string // for display purposes only
    active bool
    roles  []*Role
    groups []*Group
}
```

#### Role
A role represents a collection of privileges.  Each user in the system has one or more roles assigned.

`flow` holds the following details of each role.

```go
type Role struct {
    id    uint16
    name  string
    privs []*Privilege
}
```

#### Group
A group represents a specified collection of users.  A user belongs to zero or more groups.  Groups can have associated privileges, too.

`flow` holds the following details of each group.

```go
type Group struct {
    id    uint16
    name  string
    privs []*Privilege
}
```

#### Privilege
A privilege represents an authorisation to perform a specific action on a specified set of documents.  Privileges can be held by individual users, roles and groups.

`flow` defines a privilege as having the following structure.

```go
type Privilege struct {
    privs    []PrivilegeType
    resource string
    doc      *Document // only if not on a resource
}
```

#### PrivilegeType
Privileges are an enumerated set, and apply to resources and documents.  They closely model REST conventions.

```go
type PrivilegeType string

const (
    PrivList     PrivilegeType = "LIST"
    PrivCreate                 = "CREATE"
    PrivRead                   = "READ"
    PrivUpdate                 = "UPDATE"
    PrivDelete                 = "DELETE"
    PrivUndelete               = "UNDELETE"
    PrivArchive                = "ARCHIVE"
    PrivRestore                = "RESTORE"
)
```

#### Resource
Resources represent collections of documents of a given type.  Resources are the first line of targets for assignment of privileges.

`flow` defines a resource as having the following structure.

```go
type Resource struct {
    id        uint16
    name      string
    endpoint  string
    namespace string // optional
}
```

#### Document
Documents are central to the workflow engine and its operations.  Each document represents a task whose life cycle it tracks.  In the process, it accumulates various details, and tracks the times of its modifications.  The life cycle typically involves several state transitions, whose details are also tracked.

`flow` defines a document as having at least the following data.  Applications are expected to embed `Document` in their document structures.

```go
type Document struct {
    id       uint64      // globally-unique
    dtype    DocType
    title    string
    text     string      // primary content
    author   *User       // creator of the document
    ctime    time.Time
    events   []*DocEvent // state transitions so far, tracked in order
    revision uint16
}
```

#### DocType
A document's type is one of a set of enumerated types, but as defined by the consuming application.  `flow`, therefore, does not assume anything about the specifics of any type.  Instead, it treats document types as plain, but controlled, text.

For its purposes, `flow` requires that valid document types be available as constant strings.  An **example** is presented here.

```go
type DocType string

const (
    DocTypeIssue DocType = "ISSUE"
    DocTypeTask          = "TASK"
)
```

#### DocEvent
Together with documents, events are central to the workflow engine in `flow`.  Events cause documents to switch from one state to another, usually in response to user actions.  They also carry information of the modification to the document.

`flow` defines an event as having the following structure.

```go
type DocEvent struct {
    doc         *Document
    user        *User     // person causing this modification
    mtime       time.Time
    text        string
    newState    *DocState
    newRevision uint16    // serves as cross-check
}
```

#### DocState
A document's state is one of a set of enumerated types, but as defined by the consuming application.  `flow`, therefore, does not assume anything about the specifics of any state.  Instead, it treats document states as plain, but controlled, text.

Each defined document state has a specified set of valid succeeding states.  A document in the current state, upon the occurrence of a transition event, can switch into one of only the specified successor states.

```go
type DocState struct {
    dtype      DocType // for namespace purposes
    name       string
    successors []*DocState
}
```

#### Workflow
Workflow objects represent the entire life cycle of a document.  A workflow begins with the creation of a document, and drives its life cycle through a sequence of responses to user actions or other external events.

The engine in `flow` is visible primarily through workflows and their behaviour.  A workflow in `flow` has the following structure.

```go
type Workflow struct {
    id        uint32
    title     string
    begin     *FlowNode
    path      []*FlowNode // traversed so far, tracked in order
    completed bool
    aborted   bool
    // TODO(js): What else is needed here?
}
```

#### FlowNode
Nodes in a workflow represent processing or explicit user action.  Each node has a type, keeps track of its document, and has a registered function that - when evaluated - answers the next node in the flow.

```go
type FlowNode struct {
    doc      *Document
    ntype    NodeType
    incoming []*FlowNode
    proc     func(*Document, ...interface{}) (*FlowNode, error)
}
```

#### NodeType
From a workflow perspective, nodes are of a few different types.  Accordingly, their processing requirements are different.

`flow` models and handles the following.

```go
type NodeType byte

const (
    NodeTypeBegin   NodeType = iota + 1 // none incoming, one outgoing
    NodeTypeEnd                         // one incoming, none outgoing
    NodeTypeLinear                      // one incoming, one outgoing
    NodeTypeBranch                      // one incoming, two or more outgoing
    NodeTypeJoinAny                     // two or more incoming, one outgoing
    NodeTypeJoinAll                     // two or more incoming, one outgoing
)
```

#### Mailbox
Each user in the system and each group in the system has a mailbox that receives workflow messages.  Mailboxes have virtually unlimited size, though applications may enforce a size limit.

`flow` models mailboxes as follows.

```go
type UserMailbox struct {
    user     *User
    group    *Group
    messages []*Message
}
```

#### Message
Messages can be informational or seek action.  Each message that seeks an action contains a reference to the document that began a workflow, as well as the next node in the workflow.

```go
type Message struct {
    id       uint64    // globally-unique
    user     *User
    group    *Group
    doc      *Document // can be `nil` for informational messages
    text     string
    node     *FlowNode // always `nil` for informational messages
    sent     time.Time
    received time.Time
    seen     time.Time
}
```