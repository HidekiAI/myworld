# myworld

I am the teacher/trainer of MyWorld (and you are the teacher/trainer of your world)...

Server/client based RTS-RPG, where A.I. related integrations may involve usage of trained ML models as well as external services (such as Google Vision, etc) in which, as a solo player, you record your actions you'd want to teach, and M.L. will remember the situations (actions taken based on surroundings, environments, times, stats, etc), so that NPCs surrounding you will take similar actions when similar situations are encountered.  More you teach, better the NPC will mock and adapt.

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

3rd party services will always be based on isolated Docker container images (i.e. nginx, oauth proxy, etc), and orchestated via Kubernetes.

### Server

#### ML Models

For the sake of having a word to describe the goals, I wull use the term "Action" to represet any actions, tasks, or events that was emitted by an entity such as player, NPC, enemies/monsters, objects (vehicles, horse, pot on a fireplace), etc.

"Actions" should be 100% deterministic so that a time-window of recorded from [[T0...TN]] can be replayed for later training (that will be total of N+1 actions).

As a starter, I am thinking to just use RNN (Recurrent Neural Network) for the A.I.  This is because it is relatively simple to implement, and it is also relatively easy to train.  I am also thinking to use TensorFlow for the implementation (over Torch), as it is relatively easy to use, and it is also relatively easy to train in Rust.  It's been suggested that I should also consider GNN (Graph Neural networks), or hybrid combination of RNN and GNN, either way, here are my initial thinking/design/goal:

In a nutshell, I have the following inputs for training and predicting:

- "Action" that the player has taken - hopefully, this fits the model of fitting in an event in kafka?
- Sourroundings - as much as possible, both dynamic and static objects in the radius R.  This includes walls, trees, ground type, clock on the wall, NPCs, players, crates, etc.
- Other "Actions" - NPC, objects, monsters, etc that triggered in parallel to the player's "Action"
- Time of day - some environmental settings may get affected based on vision (sun light vs moon light), time (traffics are less during the night), etc
- Status - my HP|MP|EXP|STR|AGL|INT|WIS|CHA|LUK|etc, etc
- Plans - Last action taken (what was I doing, what should I do next)
- Inventory/Equipment/Buffs/Debuffs - what do I have, what am I affected by?
- Skills - what am I capable of?
- Relationships - who can hurt me, who can I help, who can I attack?

These should (hopefully) be generic enough that will work globally, yet whether you are actual player or NPC, should be enough to  conclude on justifications of the "Action" taken.

For reinforcement learning, I'd like to be flexible on reward function that reflects the success of an action (i.e. user-defined callbacks).

- Short-Term Rewards: Immediate feedback after each action. For example, gaining experience points, picking up items, or losing health.
- Long-Term Rewards: Evaluate the cumulative effect of actions over a longer period. For example, surviving longer, completing objectives, or helping teammates.
- Custom Rewards: Define rewards based on specific game mechanics, such as sacrificing HP for team benefit, where the reward considers the overall team success.

### Client
