# Distributed and Parallel Computing - Distributed Computing

# Introduction
- Distributed computing means computing on multiple nodes/processes
- Each node as a unique ID
- Nodes are connected via channels (i.e. edges)
- Nodes do not share memory or a global clock
- Nodes are either connected or not connected
- Networks are strongly connected (i.e. there is a path between any two nodes)
- Networks don't necessarily have to be complete (i.e. edge between every
  pair of nodes)
- Channels may be directed or undirected
  - TODO: How does this relate to "nodes are either connected/not connected"?
- Communication is passing messages over channels
- Channels are not necessarily FIFO
- Communication is asynchronous

## Parameters of the Network
- `N` number of nodes
- `E` number of edges
- `D` diameter of the network, i.e. the longest "shortest path" between any
  two nodes

## Failures
- Failure rate (FR): Number of failures per unit time
- Mean time before failure (MTBF): `1/FR`
- e.g.
  - 1000 nodes, all critical
  - MTBF of each node is 10,000
  - `FRₙ = 1/MTBF`
  - `FRₙ = 1/10,000`
  - `FRₛ = sum(FRₙ for n in nodes)`
  - `FRₛ = 1,000 * (1/10,000)`
  - `FRₛ = 1/10`
  - `MTBFₛ = 1/(1/10)`
  - `MTBFₛ = 10`

## Spanning Trees
- Spanning tree:
  - Contains all the nodes of the network
  - Edges are a subset of the network's edges
  - No cycles
  - Undirected
- Tree edges: edges in the spanning tree
- Frond edges: edges in the network but not in the spanning tree
- Sink tree: tree made by taking all edges of a spanning tree, and making them
  directed so that all paths end up at some chosen root node

### Usage
- We want to send a message to the whole network
- If we send to neighbours recursively, it doesn't terminate
- If we send to the spanning tree root, and then follow the reverse of the sink
  tree (as in the edges are reversed), then every node only sees the message
  once

## Transition Systems
- The behaviour of a distributed algorithm is given by a transition system:
  - Set of configurations: `(λ, δ) ∈ C`
    - Global state of the algorithm
    - `C` is the set of all possible configurations of an algorithm
  - Binary transition relation on `C`: `λ → δ`
    - Changes the global state from one configuration to another
  - A set of initial configurations: `I`
- A configuration is terminal if there are no transitions out of the
  configuration
- An execution is a sequence of states in `C`, beginning with an element in `I`,
  and can either be infinite or ending in a terminal configuration
- A configuration is reachable if there is an execution that contains it

### States and Events
- Configuration consists of
  - The set of local states of each node
  - The messages in transit between the nodes
- Transitions are connected with events
  - Internal: an action that modifies the internal state of a process
  - Send: A message is sent from a process
  - Receive: A message is received by a process
- Initiator: the process where the first event is an internal, or a send event
- Centralised: only one initiator
- Decentralised: multiple initiators

## Assertions
- Safety property
  - Must be true on every reachable configuration
- Liveness property
  - Must be a reachable configuration with this property, from every
    configuration

## Orders
- Total order
  - `a ≤ a` reflexivity
  - `a ≤ b && b ≤ a → a = b` antisymmetry
  - `a ≤ b && b ≤ c → a ≤ c` transitivity
  - `a ≤ b || b ≤ a` totality
- Partial order
  - `a ≤ a` reflexivity
  - `a ≤ b && b ≤ a → a = b` antisymmetry
  - `a ≤ b && b ≤ c → a ≤ c` transitivity
  -  No totality
- Causal order
  - `a ≺ b` iff `a` must occur before `b` in any execution
  - If `a` is send, and `b` is receive, then `a ≺ b`
  - `a ≺ b && b ≺ c → a ≺ c`

## Computations
- Executions are too specific, different orders means different executions but
  same work is done
- Instead, we talk about "computations", a set of executions equivalent up to
  permutations of concurrent events

# Local Clocks
- Maps to a partially ordered set (i.e. integers) such that
  - `a ≺ b ⇒ C(a) ≺ C(b)`
- Requires each process maintains a record of:
  - Its local logical clock, measuring the process's own progress
  - Its global logical clock, measuring its perception of global time

## Lamport's Clock
- Notated as `LC(event)`
- Tracks local and global logical clocks in one variable
- Let `a` be an event, and `k` be the clock value of the previous event
  - If `a` is internal or send, `LC(a) = k + 1`
  - If `a` is receive, and `b` is the send event of `a`, `LC(a) = max(k, b + 1)`
- Consistent with causality, but not strongly consistent
  - i.e. `a ≺ b ⇒ LC(a) ≺ LC(b)` but _not_ `LC(a) ≺ LC(b) ⇒ a ≺ b`

## Vector Clock
- Each process keeps a vector, the same length as the number of process
- `vᵢ[i]` is the local logical clock
- `vᵢ[j]` where `i ≠ j`, is process `i`'s most recent knowledge of process `j`'s
  logical clock
- Initialise all of `vᵢ` to 0
  - If `a` is internal or send
    - `vᵢ[i] += 1`
  - If `a` is receive, and `m` is the vector clock sent with `a`
    - `vᵢ = max(vᵢ, m); vᵢ[i] += 1`
- Ordering
  - `u = v ↔ ∀i. u[i] = v[i]`
  - `u ≤ v ↔ ∀i. u[i] ≤ v[i]`
  - `u < v ↔ u ≤ v && ∃i. u[i] < v[i]`
  - `u || v ↔ u !≤ v && v !≤ u`
- Strongly consistent!

## Mutual Exclusion
- Centralised
  - Works
  - Easy to implement
  - Fair (in order of request)
  - No starvation (no node waits forever)
  - Only 3 messages per use of resource (ask, grant, release)
  - However, single point of failure, and a big bottleneck
- Decentralised
  - To request a resource, send message to all process requesting resource
  - When process receives message:
    - If not using and doesn't want
      - Send OK back to sender
    - If has access
      - No reply, but queues the request
    - Wants the resource but doesn't have it
      - Compares logical clock value of requester
        - If requester has lower, send OK message
        - If requester has higher, queue message and send nothing
  - When sending a message, wait for an OK from everyone else, once this happens
    it can access the resource
  - When done with the resource, check queue and send OK to all of them
- Advantages: works, fair, no starvation
- Disadvantages: `2(n - 1)` messages, `N` points of failure, every node needs to
  keep track of every other node in the system

# Snapshots
- Allows saving program state so we can resume later
- Allows returning to previous state if things break
- With distributed systems, can't do this because there's no global clock
- Messages could be on the fly, difficult to record this state
- Recording local snapshots must be coordinated correctly to ensure a consistent
  global snapshot
- If each process takes a local snapshot:
  - An event is pre-snapshot if it occurs in a process before the local snapshot
  - Otherwise it is post-snapshot
- A snapshot is consistent if
  - When `a` is pre-snapshot, `x ≺ a` implies `x` is pre-snapshot
  - A message is included in channel state if its sending is pre-snapshot and
    its receiving is post-snapshot

## Chandy-Lamport Algorithm
- Applies to FIFO channel systems only
- Send control messages called markers along channels to separate pre- and
  post-snapshot events and trigger local snapshots
- Initiator takes local snapshot and sends marker through all outgoing channels
- When process `pₘ` receives marker along channel `cₙₘ`
  - If `pₘ` has not yet saved state
    - `pₘ` saves local state
    - `pₘ` sets `cₙₘ` state to `{}`
    - `pₘ` sends marker through to all outgoing channels
  - Else
    - `pₘ` records state of `cₘₙ` as set of all basic messages received after it
      has saved its local state, and before it received the marker message from
      `pₙ`

### Correctness
- If `a ≺ b` and `b` is pre-snapshot then `a` is pre-snapshot
- If `a` is send and `b` is receive of the same message in processes `p` and `q`
  - `b` is pre-snapshot → `q` has not received a marker when `b` occurs
  - Since channels are FIFO, `p` has not sent a marker when `a` occurs
  - Hence, `a` is pre-snapshot
TODO: Rest

## Lai-Yang-Mattern Algorithm
- Works on non-FIFO channels
- Rather than having separate marker messages, attach boolean flag to basic
  messages
  - Typically described as white/red
- Lai-Yang algorithm didn't need control messages, but required keeping all
  message history
- Lai-Yang-Mattern algorithm uses control messages with logical clocks

### Algorithm
- Every process initialised to white
- When a process saves its local state:
  - Turn red
  - Send control message on all outgoing channels to say how many white messages
    it has sent down that channel
- Every basic message is the same colour as the process that sends it
- White process can save local state at any time
  - But must save it no later than on receiving a red message, and before
    processing that message
- When receiving the control message:
  - Save local state if it hasn't already
  - Process knows how many white messages it has received currently on each
    input channel
  - Process knows how many white messages it needs to receive from the control
    message
  - Waits for white messages
  - Each process channel computes channel state as the set of white messages it
    receives after saving its local state

## Mutliple Snapshots
- Instead of red/white, use counter `k`
- On first snapshot, `k = 0` is white, `k = 1` is red
- On second snapshot, `k = 1` is white, `k = 2` is red
- If two nodes start snapshot concurrently, they will both increment and the
  snapshot will be the same
