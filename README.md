# myworld

I am the teacher/trainer of MyWorld (and you are the teacher/trainer of your world)...

Server/client based RTS-RPG (or maybe even TD), where A.I. related integrations may involve usage of trained ML models as well as external services (such as Google Vision, etc) in which, as a solo player, you record your actions you'd want to teach, and M.L. will remember the situations (actions taken based on surroundings, environments, times, stats, etc), so that NPCs surrounding you will take similar actions when similar situations are encountered.  More you teach, better the NPC will mock and adapt.

## Infrastructure

- [Server](#server):
  - Microservice oriented, orchestrated via Kubernetis
    - S2S
      - via Protobuf gRPC for interally created services
      - else it differs i.e. REST with XML, REST with JSON, async via kafka (not using Rabbit), or even SOAP XML (XML is expensive, so only for rare occasions)
      - S2S via REST for external services (i.e. Google Vision, etc)
      - S2S via UDP for gameplay (i.e. game state, etc)
      - S2S via TCP/IP for gameplay (i.e. chat, etc)
    - C2S via either REST (via nginx) or gRPC
      - Still debating whether both TCP/IP and UPD are needed
    - OAuth2?
    - Headless
- [Client](#client):
  - C2S - see server side C2S
  - Multiple UI presentations
    - TUI
    - GUI

## Gameplay

The more you train, the more the NPCs will bias towards your actions.  In a tutorial, you're forced to record the basic "common sense" actions, so that you can observed the NPCs to take same actions as you've recorded.  Of course, what you record could differ between each players, mainly because common sense to one person may not be "common" to another.  Some may take the efficient quickest approach, some may take longer easy-going approach, as long as you get the goals accomplished, that's all it matters.

In terms of reinforced learning, what it learns is what you've recorded, which it assumes is always the correct and expected goals, hence ML is rewarded as long as it gets to the final goal regardless of how long it took, preferably within [[T1..TN]] turns (if it takes longer tan TN turns, the reward will be gradually reduced, and if it accomplished the goals quicker than TN, the reward will be gradually increased).  I.e. on time by TN turns, then +0.45 points, earlier than TN turns is +0.5 points, and later than TN is 0.0 points.  And then, on top of that, we always add +0.5 as long as goals are accomplished, so as long as you get 0.5 or greater, you've accomplished the goals.  The separtions of accomplishing the goals and bonus for finishing on or earlier than expected turns, is because we want to reward as long as goals are reached.  For example what you have taught (recorded) was riding a bicycle on a flat terrain, and it took N turns to travel X distance; but what the A.I. was trying to accomplish was on a icy-up-hill in which it took significantly longer turns to travel the same distance, yet A.I. was rewarded close to nothing would be unfair, hence as long as A.I. gets rewarded 0.5 or greater, it's a success.  Perhaps, if there are ways to measure how close the A.I. got to the goals before it died (HP = 0), then we can reward less than 0.5 points, but that's still under considerations since there's no sure ways to say "that was close to accomplishing the goals".

TBD - I personally like city-builder and/or factory kind of RTS and Tower Defense, but that's still under considerations...

## Development

As much as possible, most logics are written in the libmyworld library, in which tests, examples, and benchmarks are written within.  Where possible, it will all be in Rust, but where 3rd party libraries are not available, C++ (kafka-streams) will be used as 2nd priority, then Go (kafka-streams - note, they also have Java), then Python.  Note that if there are choices between C++ and Go (or other memory-managed and/or GC based languages), that will have to be chosen over C++.

3rd party services will always be based on isolated Docker container images (i.e. nginx, oauth proxy, etc), and orchestated via [Kubernetes (because it's proven and debugged)](https://cloud.google.com/blog/products/containers-kubernetes/bringing-pokemon-go-to-life-on-google-cloud) (most devs decide to consider kubernetes when load-balancer (scaling) is anticipated).

Side note: Though I may mention other languages such as C++ and Java (or even the dreadful Python for ML), ideally most of my implementations will be in Rust.

### Server

Firstly, the network protocol between client-to-server will be REST based.  The server-endpoint API serving host will just be a proxy to relay request to a micro-service.  This should make it flexible to easily swap/switch this proxy (micro) service to be HTTPS, HTTP, nginx, IIS, Apache, etc, and completely decoupled.  The relay logic to other micro-services can be written in Java, Go, Rust, or Python, or any Protobuf gRPC supported languages of choice, but for security reasons and being the first line of defense, ideally it *SHOULD* be written in Rust.

The server-side micro-service will be written in Rust and will be stateless, and the REST proxy will use gRPC Protobuf to communicate with the micro-service.  In terms of stateless, state (snapshots) are persisted in cache micro-service (Redis) and/or database (Postgres).  The micro-service will be orchestated via Kubernetes.

In past practice, I've been used to passing updated/altered state in-memory from one module to another, in which the state is only read in once on request, and written/updated on successful completion of that request.

I think it's perfectly fine for modules to pass updated states from one to another, but I disagree on the part where I only update when all transactions succeed (such as updating database account balance).  I think it's better to update the state in-memory first, and then update the database asynchronously.  In the past company I was employed with, they did not wish to post the database which contained player account balance until entire completion of the request/response.  The main requirement from both DBA and S.E. was because the fact that they wanted to reduce the number of visits and updates to the TSQL database to just twice per request (once on request, and once on 200 response).  There's nothing wrong with this requirement, especially if one is trying to achieve 1000s of requests per second, and there are millions of user accounts.

The whole issue is decisions on when to update the persisted storage and assume that each completion of successful update is final and cannot rollback, in which means the entire request/response has to be treated as a single atomic transaction, rather than two-phase commit (2PC).  The obvious problem we begin to picture is MUTEX situation of unordered updates.  Couchbase for example, has a concept of "last-write-wins" (LWW) which is a form of optimistic concurrency control (OCC).  This is a good solution for most cases, but it is not a good solution for financial transactions.  In the case of financial transactions, we want to ensure that the order of updates is preserved, and that the final state is the correct state.

There is a design in which an architect proposes that they approach the way Starbucks approaches the way a customer pays for the coffee at the register, takes the money, something goes wrong, and the coffee cannot be served, hence the customer is refunded.  This is similar approach all businesses follow including banks.  There is a post in which a transaction will (optomistically) be assumed to be successful, and if it fails, start a new/next transaction to refund, but it's not considered as failure in a sense, but just another transaction called refund.  When I use an ATM card to pay gasoline (I rarely do, almost never), my wife will notice a transaction from a bank in which it took -$1.00 from our account, and immediately refunded back +$1.00 to our account.  This (I assume) is to verify that you have a valid (unexpired, usable, etc) card that the bank that backs our ATM card that my wife allows me to carry around for emergency.

And to remain in sequential even in an unordered requests, one solution is to version and timestamp as safeguards.  This won't happen on a gas-pumps because I only have two hands, but maybe there was an edge-case bug in which I could begin pumping gas while the pump was contacting our bank to see if our ATM card is valid, and maybe the developer felt it was a feature and an optimization by allowing up to $1.00 of petro/gas to be pumped before getting confirmation from the bank is successful, mainly because it's only $1.00... (Remindes me of "Superman III" where Richard Prior's character acquires $0.005 (1/2 cents) from each account.)

But what happens if there was a disconnection right after it took my -$1.00 from our account?  My wife will now make me call the bank, in which I'd imagine the bank will tell me that the gas-station is whom I have to call.  What a nightmare!  Long story short, to avoid this nightmare, we want design systems that will not post update until all is successful.  But does that mean that the gas station should not validate my ATM card until I finish pumping the gasoline?  What I just wanted 5 liter but the pump run out at 3 liter?

One solution is record in sequenced order (version and timestamps) as an eventual consistency, mainly because I only have 1 ATM card, and it's considered to be an unexpected/unhandled exceptions failure/panic if somebody else has a same exact ATM card as mine, and that identity-thief used our ATM card the same time the gas pump was trying to validate my card (from bank's point of view, it doesn't know whether that theif in California or myself in Texas are the real owner of the card).  It just knows that there was a concurrency problem in which it is assume that it should have never happened.

Finally, if asynchronous state updates to persistent storage is a must in the design, there should never be in-memory state ever.  One should never pass from module to module the updated states, but rather per module it should read the current snapshot of states, and upon exit/completion of the module, it should write back if the state was altered.

For a longest time, we used to use the term "Idempotency" but at my last company I've worked for, they used the phrase "recovery", but regardless of how to phrase it, if the design was to only collect the state from the persisted state just once, adjust/update states and pass the updated states to the next module, and then write back the updated states to the persisted state only on 200 response, if there are disconnects anywhere in the middle, the system should be able to recover from the last known good state.  Obvious caveat that was treated as an edge-case was even though it was anticipated as 200 response, the TCP/IP connection trying to send that 200 back never gets to the client, in which the player will assume 500 response and will try to resend the request, which the server then is now out of sync because it was anticipated as 200 response.  To solve this, the pattern for the solution was to distinguish the differences between important response that required confirmation (a receipt) and ones that did not.  Obviously, some may argue that there is a design flaw here, because if the client side was also stateless, all it had to do request the backend for current state snapshot prior to making next request that will possibly alter to new state.

An experienced backend engineers would obviously argue that we want to bundle as many roundtrips as possible, I've onced worked on a peer-to-peer design in which by applying [Nagle's algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm), we gained about 10-15 FPS in performance!  Though the people I've worked with at my former (and one before that) comprehends the signficances of Nagle's algorithm, they did somehow concluded that they'd want to keep states on the client side to avoid unnecessary roundtrips by keeping in-memory states based on last response message.  This required every responses to return delta or absolute changes between last and new states, but often times, because backend developers do not know what data the frontend developers need other than the obvious data requested from the endpoint request (i.e. if the endpoint route was "get_player_inventory", backend engineers would never think to return player's position) because (at least, to me) REST is all about being specific on request and response.  In any case, rather than having player request more than once, we tend to bundle as much data as possible into single response so that client do not need to request multiple times.

Going back to the issue of different patterns used, upon quick research, I've found the following:

- Saga Pattern: A sequence of transactions that can be undone if one part fails.  Each step has a compensating action to revert the changes made by previous steps if needed. This pattern is suitable for long-running transactions and distributed systems.
- CQRS (Command Query Responsibility Segregation) Separation of Commands and Queries: Commands update the state, while queries read the state.  This can be combined with event sourcing where state changes are stored as a sequence of events, allowing for precise tracking and rollback if necessary.
- Event Sourcing: Instead of updating the state directly, events that represent changes are stored. The current state is derived by replaying these events. This approach provides an audit trail and makes it easier to handle failures and rollbacks.
- Distributed Transactions:
  - Two-Phase Commit (2PC): Ensures that all participants in a transaction agree on the commit or rollback. Useful for maintaining consistency across multiple systems but can impact performance.
  - Three-Phase Commit (3PC): A more complex variant of 2PC that adds a prepare phase to reduce the likelihood of a transaction being stuck in an uncertain state.

 In terms of rollbacks, reverting, and failures, one thing I do like are API's which takes callbacks and/or either configurations or parameter indicating strategies on failure when response messages are not able to be sent back.  On that note, we assume all S2S connections are TCP/IP stream based.  And by assumptions, REST communications between C2S is also TCP/IP stream based.  That had to be mentioned because unlike UDP, at the application layer (Layer 4), you can detect network disconnection (usually connection timeout on abnormal disconnection).  Note that I do not know whether gRPC uses `boost:asio` (i.e. Winsock IOCP), but (for example) with 1000 clients, there is 1000 TCP/IP sockets, but if there are ways you can send to each TCP/IP connections a keep-alive/heartbeat messages on an idle connection which backend have not heard in a while, one can also detect abnormal disconnection.  This is because in most cases, we end up detecting disconnection when we try to send (stream) to it and get an I/O error.  TCP/IP's characteristics are to try to reconnect if disconnected dating back in very old technologies of MODEM 300 bauds, so TCP/IP timeouts are usually long time (Windows used to default to 600 seconds), but usually, when attempt to send, you can get back immediate failure notice (I do not know how it is today, but one experiment I've done in the past was when differences between unplugging the ethernet cable on the NIC versus unplugging it on the hubs and switches, as well as unplugging the power of the hub/switches - the results were different).

Whether the micro-service rolls-back on its own or not differs and/or depends on situations as well as how you use it.  For example, suppose you have PostgreSQL as your database that is contained in a Docker container.  The question is, do you access that PostgresSQL directly port 5432, or do you write a proxy service which resides inside the same container, and you'd query that proxy service instead, in which it has a C# `try/catch` statement that inside the `try` block, it will attempt to send result-rowsets back, but if that result-rowset does not end back to the calling microservice, the `catch` block will automatically rollback.

Or in contrast, as a caller/user, you have your own `try/catch` and you detect TCP/IP timeout, and record that you need to rollback on reconnect.  But the problem with the later is that you don't know why you got network disconnections (suppose you are disciplined enough to have multiple `catch` statement, one for transaction exceptions, and another for network exceptions), but you don't know whether the whole transaction was successful, but you got disconnected prior to getting the rowsets back, or not.  At least for the former approach, having multiple `catch` blocks or even nested-try-catch, one can detect which part of failure it was with and internally know whether to rollback or not.  This is possible as long as we assume that loopbacks, localhosts, and named-pipe communications are always reliable, and that only point of failure is that the entire host (container) is hosed...

There are pros-and-cons on both approaches, and cons on the approach of the proxy-service in the container is that you now have to write another service and make sure to remember to build a container with it as part of the container.  And of course, another obvious cons is more moving parts, more potential point of failures (you had a watchdog that was watching the health of PostgresSQL, but now, you have to also watch whether the proxy service is also healthy).  One may have assumed that it's more fault-tolerant having some proxy service rollback for us because it's more aware of what failed, but it could backfire in other ways (i.e. maybe your proxy-service binds to 0.0.0.0:4321 and you have a bug, or the library you use has a bug which it won't release that port 4321, hence when you tried to restart the service, it kept on failing and you couldn't figure out why because `netstat` shows that your service is the listener of that port, but you don't realize it's the old instance that crashed, not the replaced binary, and so on...)  Of course, I'm over exagerating, but you get the point.

#### ML Models

For the sake of having a word to describe the goals, I wull use the term "Action" to represet any actions, tasks, or events (associated within the "topic") that was emitted by an entity such as player, NPC, enemies/monsters, objects (vehicles, horse, pot on a fireplace), etc.

"Actions" should be 100% deterministic so that a time-window of recorded from [[T0...TN]] can be replayed for later training (that will be total of N+1 actions).

As a starter, I am thinking to just use RNN (Recurrent Neural Network) for the A.I.  This is because it is relatively simple to implement, and it is also relatively easy to train.  I am also thinking to use TensorFlow for the implementation (over Torch), as it is relatively easy to use, and it is also relatively easy to train in Rust.  It's been suggested that I should also consider GNN (Graph Neural networks), or hybrid combination of RNN and GNN, either way, here are my initial thinking/design/goal:

##### Cause and Effect

The way I see it, it's all about predicting the future (next best event-turn) based on past event(s).  From what I comprehend on my small brain, this sequence of events or time-based-series is the theme of recurrent neural network, and that these routes of decisions are hidden layers in NN, and each nodes in the layer are combinations of different parameters such as status and relationship.  And as I (try to) comprehend it, each hidden layers remembers (persists) the weight (result) and are stored as a memory (so to speak) as hidden state, so that next time it needs to identify the similar situations, it can recall the weight (result) from the memory (hidden state) and use it to predict the next best event-turn.  Wait, but don't we just store the input and final output, and don't care what the hidden-layers states/weights are (because it's deterministic as long as last time it was trained the input and hidden layers have not changed, added, restacked, altered, etc)?

In a nutshell, I have the following inputs for training and predicting:

- "Action" that the player has taken - hopefully, this fits the model of fitting in an event in kafka?
- Sourroundings - as much as possible, both dynamic and static objects in the radius-R.  This includes walls, trees, ground type, clock on the wall, NPCs, players, crates, etc.
- Other "Actions" - NPC, objects, monsters, etc that triggered in parallel to the player's "Action"
- Time of day - some environmental settings may get affected based on vision (sun light vs moon light), time (traffics are less during the night), etc
- Status - my HP|MP|EXP|STR|AGL|INT|WIS|CHA|LUK|etc, etc
- Plans - Last action taken (what was I doing, what should I do next)
- Inventory/Equipment/Buffs/Debuffs - what do I have, what am I affected by?
- Skills - what am I capable of?
- Relationships - who can hurt me, who can I help, who can I attack?  It will become super complicated when there are allies of different stats, enemies of different stats, and trying to make a decision (action) based on closest enemy (i.e. what if closer one has 100% HP, but the one further away has 1% HP?).  So this parameter should be based on callback/delegate/lambda-functions defined customised.

These should (hopefully) be generic enough that will work globally, yet whether you are actual player or NPC, should be enough to  conclude on justifications of the "Action" taken.

For reinforcement learning, I'd like to be flexible on reward function that reflects the success of an action (i.e. user-defined callbacks).

- Short-Term Rewards: Immediate feedback after each action. For example, gaining experience points, picking up items, or losing health.
- Long-Term Rewards: Evaluate the cumulative effect of actions over a longer period. For example, surviving longer, completing objectives, or helping teammates.
- Custom Rewards: Define rewards based on specific game mechanics, such as sacrificing HP for team benefit, where the reward considers the overall team success.

### Client

## Journal

You can skip this section, I'm just trying to keep journal/logs of what it takes for a one-man developer to fit all this together, useful to gauge time-frame when contracting, I can give an estimates of how much work and time was involved on each epic stories that I'd call "phases".

### Phase 1: Setup and System Requirement Gathering

In the very first phase, I need to gather informations about:

- Storage: What are my persistent storage?
- Performance: How many round trips per second?  How fast do I need to complete each round trip?
- How parallel do we need to be?

I need to at least get Kubernetes setup and have Apache-kafka as a pub/sub broker for actions (so for now, just one kafka, and probably next one will be a broker specific to user authentications) and Zookeeper.

#### Parallellism of Processes

All in all, at least as a starter, when it comes to "how parallel", I'm trying to solve that by using (1) Kafka to allow scalabe pub/sub concurrently, (2) Kubernetes to easily scale up/down the number of pods, and (3) Docker to isolate deployments of what we dev in its own container.  I'm not sure if I'm over-complicating it, but I'm trying to make it as scalable as possible.  I'm not sure if I'm over-complicating it, but I'm trying to make it as scalable as possible.

By enforcing what we develop to be self-contained in its own container, it will force me to continously think about:

- How to make it as stateless as possible (i.e. no persistent storage)
- How to make it as loosely coupled as possible (i.e. no direct dependencies)
- How to make it as scalable as possible (i.e. no single point of failure)

#### Storage

Probably as the project grows, there will be requirements of new storage, but for now, I'm trying to keep it as simple as possible.  I'm thinking of using:

- Kafka for message queuing for 1-to-1, 1-to-many, many-to-1, many-to-many communication between microservices.
- Redis for caching meta-data for real-time updates and inquries, mainly ping-pong between client and server, and for real-time updates of game state (i.e. player's position, HP, MP, etc)
- PostgreSQL for persistent storage for mainly player account

Probably will not get to Redis and PostgreSQL for a while, but I think it's good to declare ahead of time on storage requirements.

I'm still trying to learn where I can store training data, and where I can store trained models.  I'm thinking of using [MinIO](https://github.com/minio/minio-rs) for storing training data and trained models, but because it's biased towards [Amazon S3](https://min.io/product/s3-compatibility), I'm not too sure if it's the right choice.  I'm still trying to learn more about it.  For now, I'm just going to use local storage for training data and trained models.

#### Performance

For this application, it is assumed to be a solo (single) player game, and the backend is mainly separated into microservices for the sake of machine learning to occur on a separate host (but it's possible to run all on local machine for dev purposes).

But whether it is single client or multiple, firstly what is still of a concern is roundtrips per second (disregarding latency, but only at the level of client-to-server, and what goes on in the service-to-service, etc).

#### Hindsights and Postmortems

It was so trivially simple to install and setup minikube in Debian, only tricky part is not being able to sniff (via `tcpdump`) network traffic behind the Docker and Docker-Bridge NATs, that it was a 2-day long journey of painful learning that I should *NEVER* install Kubernetes on Windows.  I've tried many approaches:

I've did not realize how painful it is to setup (not install) Kubernetes on Windows.  I've tried many approaches:

- Docker (WSL2 based) - I do not know what or where it went wrong, but I thought this will be my preferred way, and I went ahead and (re)installed Debian WSL2 first, then run the installer and told it to use WSL2 as my VM.  In the end, I do not know whether it worked or not, I could not figure out how to query Kubernetes via `kubectl` (in fact, I did not know whether I could access it via CLI since I thought I needed to install `minikube`, etc)
- Docker (Hyper-V based) - I figured at least having it run on VM and control it via Hyper-V Manager would make sense, so I tried that, but again I could not make heads-or-tails on how to connect to it (is it ssh? I could not find NAT to bridge/route to it, etc)
- Install Docker in WSL2 (Debian) - this method was a shock to learn that minikube won't run if it detects (i.e. via `nproc`) you have explicitly set your `.wslconfig` with `processors=1`, so make sure you set it to at least `processors=2`.

In the end, though I kept the WSL2 (Debian) installed, I uninstalled Docker and everything associated to it on my Windows (and disabled Hyper-V, etc).  It's partially relavent to complain, but on my laptop, I do not wish to run any VMs and other memory-hogging background services/applications on it, but my `.wslconfig` is preallocated at most 2G (to be generous).  But it turns out when you `minikube start`, it will tell you that:

```text
$ minikube start
😄  minikube v1.33.1 on Debian 12.5 (amd64)
✨  Using the qemu2 driver based on existing profile

🧯  The requested memory allocation of 1973MiB does not leave room for system overhead (total system memory: 1973MiB). You may face stability issues.
💡  Suggestion: Start minikube with less memory allocated: 'minikube start --memory=1973mb'

👍  Starting "minikube" primary control-plane node in "minikube" cluster
🏃  Updating the running qemu2 "minikube" VM ...
```

On my laptop, WSL2 alone when instanced, will eat up about 2GB (I have my local `.wslconfig` file, I think default is set to eat more than 2GB).  That's 2GB of RAM I could have allocated for language-servers and static-analysis plugins (both Vim and VSCode) such as Rust-analyzer, F# Ionide, and C# OmniSharp.  Since most of my portable libraries I use and write are compiled using `gcc` under MinGW-w64 (and though it irks me a bit due to collisions sometimes, git-for-Windows also relies on MinGW), so I do NOT need WSL2.  Yes, compared to dual-booting, WSL2 may seem like a great invention, but from a many-years Linux users who's used to QEMU and KVM, Xen, and even headless-VM based that allows easy/trivial storage sharing such as VirtualBox (headless), WSL2 is still undesirable because the file system access priveleges is completely incompatible.  When you try to access Windows files from WSL2, it'll treat the FS as NTFS/DOS, but the problem is that it usually sets the file to `0777` (`g+rwx`) so newer Windows applications that are more security aware will fail, or in my case, running `sshd` will refuse because the `.ssh` dir (and key files) are not `0600` (or just `r`) everytime I try to edit `sshd_config` file or something.

If I recall, WSL2 will assume that local FS is all `vfat` (well, I'm sure it's something else, but from `/etc/fstab` point of view, I think it is represented as `vfat`), which will allow Windows side to be able to access the files on the WSL2 side, but the problem again will occur if you try to create a file on the WSL2 side, because WSL2 has no clue what the `uid` (and `gid`) is (i.e. cannot find lookup in `/etc/passwd` and/or `/etc/group`), things starts to go wonky.  Cygwin for example, comes with a tool called `mkpasswd` that will allow you to create a `/etc/passwd` and `/etc/group` file that is compatible with Windows, but I'm not sure if WSL2 has something similar.  MinGW on the other hand, is really just a native Windows applications, so needs none of that.  With all that FS nightmare, I usually stick with MinGW.  I used to be Cygwin user, but when MinGW introduced `pacman`, I no longer needed Cygwin (only issue I still have with MinGW is that they offer 3 types of libs, but I've written bash-scripts to only install the lib files I use, so that `CMake` can find them).  Another gorgeous thing about MinGW is the fact that on Debian side, you can cross-compile and build MinGW targets!  So why do I need WSL2?

In any case, this is my long-winded way of saying that if you have a separate host running 100% Linux which you have access to always, DO NOT INSTALL anything other than MinGW on your Windows machine, and realize that WSL is only useful if you remain to practice the pattern of "I have a Windows and I run QEMU-like VM (such as Virtualbox or Hyper-V) called WSL which I easily share files without using `ssh` (actually, `scp`) to conviniently copy (duplicate) files so that I can test it on Linux host".

I have learned that `kubectl` has port-forwarding that resembles `ssh -L` (bind address/port on local side) and `ssh -R` (bind address/port on remote side), so I can use that to connect to Kafka and Zookeeper via `kubectl --address 0.0.0.0 port-forward kafka-0 9092:9092 -n kafka` and `kubectl --address 0.0.0.0 port-forward zookeeper-0 2181:2181`.  One of the reason behind this is the fact that my Linux server is multi-homed (originally it was setup for one interface for WAN and another for LAN to simplify firewall rules, but because most of my servers are now on Google Suite/Cloud, both NICs are used for LAN only (one for `eth` based subnet and one for `wlan` based subnet))

In conclusion, Kubernetes is NOT something you want to consider if you're on Windows!  Rather than spending hours and hours on trying to fit Kubernetes to Windows, spend that time creating a VM or a real host running Linux (with good network performance, disk performance, and enough RAM) and install Kubernetes there.

#### References

- [How To Deploy Apache Kafka With Kubernetes](https://dzone.com/articles/how-to-deploy-apache-kafka-with-kubernetes)
- [minikube start](https://minikube.sigs.k8s.io/docs/start/)

### Phase 2: Subscribe to Kafka and reshape data used for training

Now that Apache-kafka is accessible (tested pub/sub via `kcat`), I need to hand-generate data-set that mocks what would be generated for recording actions, and produce models so that I can have it predict my actions.  All this will be text-based and does not require any fancy GUI.  In fact, I do not think I'd need a TUI-based client, I can probably do all this in unit-test `tests` folder or `examples` folder.
