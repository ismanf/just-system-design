# Design an Elevator System

> **TL;DR** — An elevator system is **an OOD + scheduling problem** in disguise. You design the **classes** (Elevator, ElevatorController, Request, Floor, Direction) and the **dispatch algorithm** (SCAN / LOOK / nearest-car / destination-dispatch). The hard part is the scheduler — modern buildings use **destination-dispatch** where you press your destination floor at the lobby and get assigned an elevator that's optimized across all riders, dramatically reducing wait times. As a system question, the network of buildings (Otis / KONE / Schindler IoT platforms) adds **fleet management, predictive maintenance, and remote diagnostics**.

---

## 1. Requirements

### Functional
- Multiple elevators serving multiple floors.
- Riders press button on a floor (call) and inside the car (destination).
- System routes elevators to satisfy requests with low wait time.
- Open/close doors safely.
- Emergency mode (fire alarm → all elevators to ground).
- Modern: destination-dispatch input.

### Non-Functional
- Average wait time < 30 sec (typical for office buildings).
- Safety-critical (door interlocks, weight sensors, etc.).
- Real-time control loop.

---

## 2. Domain Model (OOD)

```
class Elevator:
    id
    current_floor: int
    direction: UP | DOWN | IDLE
    state: MOVING | STOPPED | DOOR_OPEN | MAINTENANCE
    target_queue: sorted list of floors
    capacity: max_persons
    current_load: persons or kg

class Request:
    floor: int
    direction: UP | DOWN          # hall call
    destination: int (nullable)   # car call or destination dispatch

class ElevatorController:
    elevators: list[Elevator]
    pending_requests: queue

    on_request(req):
        # pick elevator, append to its queue
        ...

    on_floor_reached(elevator, floor):
        # decide stop / continue / reverse
        ...
```

---

## 3. Dispatch Algorithms

### 3.1 FCFS (First Come First Served)
Naive. Same elevator handles requests in arrival order. Inefficient.

### 3.2 SCAN / LOOK (the elevator algorithm)
Each elevator services all requests in current direction, then reverses.
- LOOK = SCAN that reverses early if no more requests in the direction.

Classic example from OS literature for disk scheduling.

### 3.3 Nearest-Car
For each new hall call, pick the elevator that can serve it fastest given its current state.

### 3.4 Destination-Dispatch
At the lobby (and sometimes each floor), riders press destination floor before boarding.
- Controller knows exactly where each rider wants to go.
- Assigns elevators to groups going to same/nearby floors.
- Reduces stops per trip, increases throughput dramatically.

Modern high-rise buildings universally use destination dispatch.

---

## 4. Multi-Elevator Coordination

For N elevators in a building, the controller decides which elevator takes each call.

Optimization criteria:
- Total wait time.
- Total ride time.
- Energy consumption.
- Load balancing across cars.

Heuristic scoring: for each elevator, compute "estimated time to serve" given its current queue. Assign request to lowest-cost elevator.

For destination-dispatch, more sophisticated grouping algorithms (similar to vehicle routing).

---

## 5. State Machine

Each elevator runs a state machine:

```
IDLE → MOVING → DECELERATING → STOPPED → DOOR_OPENING → DOOR_OPEN
                                                            ↓
                          ← DOOR_CLOSING ← ← ← ← ← ← ← ← ← ←
```

Hard real-time: must respond to sensors (door obstruction, weight, emergency button).

---

## 6. Safety Interlocks

Hardware safety, but software respects:
- Door cannot open between floors.
- Cannot move with door open.
- Over-capacity warning.
- Emergency stop button.
- Fire mode: all cars to ground floor, doors open.

These are hardware-enforced ultimately; software shouldn't be the only line of defense.

---

## 7. Modern IoT / Cloud Layer

Otis, KONE, Schindler offer cloud-connected elevator fleets:
- Real-time telemetry (sensor data, run cycles).
- Predictive maintenance (vibration patterns predict bearing failure).
- Remote firmware updates.
- Building integration (access control, traffic patterns).

Architecture: edge controller on-site + cloud platform. MQTT/AMQP for telemetry.

---

## 8. Traffic Patterns

Buildings have predictable patterns:
- **Morning up-peak**: lobby to office floors.
- **Lunch**: bidirectional, lobby + cafeteria floors.
- **Evening down-peak**: office floors to lobby.
- **Interfloor**: scattered.

Optimal dispatch differs per pattern. Modern systems detect and adapt.

---

## 9. Throughput

Building elevators are sized by **handling capacity** — riders per 5-minute peak.

Formulas (engineering):
- HC = (300 × car capacity × N elevators) / RT
- RT = round trip time (depends on floors, stops, dispatch)

System design rarely goes into this depth, but worth knowing the constraint exists.

---

## 10. Common Mistakes (System / OOD)

- **One queue per elevator, no global scheduling** — uneven utilization.
- **No destination-dispatch consideration** — major performance miss in high-rise.
- **Floors above and below treated as separate cases** — code duplication. Use signed direction.
- **No safety modes** — fire emergency etc.
- **Synchronous request handling** — UI lags. Use events.

---

## 11. Cheat Card

```
PURPOSE    Route elevator cars to satisfy floor + destination requests efficiently.

CORE       SCAN/LOOK for simple buildings
           Nearest-car or scored dispatch for multi-car
           Destination-dispatch for high-rise (riders input destination at hall)
           Per-elevator state machine for door / motion
           Safety interlocks always hardware-backed

OOD        Elevator (state, queue), Controller (assignment), Request (call/destination)
            Strategy pattern for dispatch algorithm

IoT        Cloud telemetry + predictive maintenance + remote firmware

PITFALLS   per-elevator FCFS, no destination dispatch,
           no traffic-pattern adaptation, no safety modes.

RULE       The hard part isn't motion. It's scheduling.
```

---

## Resources

### Articles
- "Destination Dispatch Elevators" — KONE / Otis / Schindler white papers
- "Vertical Transportation Handbook" — George Strakosch (the bible)

### Documentation
- **MQTT** — IoT messaging protocol
- "Elevator Algorithm" — OS textbooks (disk scheduling parallels)

### Books
- *Cracking the Coding Interview* — elevator OOD problem
- *Vertical Transportation: Elevators and Escalators* — Strakosch & Caporale

### Videos
- ByteByteGo: OOD interview topics
- KONE / Otis product demos (real-world systems)

### Adjacent reading
- [Parking Lot System →](./parking-lot.md)
- [Ride-Sharing Matchmaking →](./ride-matching.md) (scheduling parallels)
- [Job Scheduler →](./job-scheduler.md)

---

*Previous:* [← Parking Lot System](./parking-lot.md)  |  *Up:* [README ↑](../README.md)
