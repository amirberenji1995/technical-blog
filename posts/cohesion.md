# 🧼 7 Shades of Cohesion: Learn to Make Your Code Do One Thing Well!

Writing clean code isn’t just about using the right syntax or tools — it’s about structure. One of the most essential (and overlooked) principles behind good software design is:

> ### **cohesion**: *how well the pieces of a module or class belong together.*

In this post, we’ll walk through **seven classic levels of cohesion**, from worst to best, using a simple scenario. You'll see how code can *work* but still be fragile, unclear, or bloated — and how small structural changes can massively improve clarity, testability, and reuse.

#### 📢 Attention! This post is co-authored by GPT-4o from OpenAI.
---

## 🚕 Scenario: Ride-Sharing Request Handling

Imagine you're building a backend system for a ride-sharing app. When a rider requests a ride, a few things need to happen:

* The ride needs to be published (made visible to nearby drivers).
* The request should be matched with an available driver.
* Notifications must be sent, and statuses must be updated accordingly.

We'll model the system using `pydantic` data classes to represent the core entities:

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class RiderSchema(BaseModel):
    id: int
    name: str
    location: str

class DriverSchema(BaseModel):
    id: int
    name: str
    location: str
    is_available: bool

class RideRequestSchema(BaseModel):
    id: int
    rider: RiderSchema
    destination: str
    request_time: datetime
    status: str  # e.g., "pending", "accepted", "cancelled"
```

We’ll now implement several versions of the `RideRequestHandler` class to reflect **each level of cohesion**, showing what it looks like in practice — and why it matters.

---

## 🧨 1. **Coincidental Cohesion** *(Worst)*

> *“Thrown together with no relation at all.”*

```python
class RideRequestHandler_Coincidental:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def publish_ride(self):
        print("Publishing ride...")

    def reset_admin_password(self):
        print("Admin password reset.")  # completely unrelated

    def clean_cache(self):
        print("Cache cleaned.")  # has nothing to do with rides
```

🧯 **Problem**: The methods serve entirely different concerns. This class is just a grab bag of logic.

---

## 🧠 2. **Logical Cohesion**

> *“Grouped because they do similar things — not because they should live together.”*

```python
class RideRequestHandler_Logical:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def send_ride_notification(self):
        print("Sending ride notification...")

    def send_promotion_email(self):
        print("Sending promotion email...")

    def send_internal_metrics(self):
        print("Sending metrics to admin...")
```

🔍 **Problem**: All methods “send” something — but the motivations and domains are unrelated (rider ops, marketing, and internal tooling).

---

## ⏱️ 3. **Temporal Cohesion**

> *“Things that happen at the same time.”*

```python
class RideRequestHandler_Temporal:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def init_status(self):
        self.ride_request.status = "pending"

    def notify_ops_team(self):
        print("Notifying ops...")

    def preload_driver_list(self):
        print("Drivers preloaded.")
```

🧭 **Problem**: These methods happen to run at the same point in time (e.g., ride initialization) — but they aren't strongly related in purpose.

---

## 🧾 4. **Procedural Cohesion**

> *“Grouped by execution flow — not semantic unity.”*

```python
class RideRequestHandler_Procedural:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def process_new_request(self):
        self.set_initial_status()
        self.log_request()
        self.trigger_broadcast()

    def set_initial_status(self):
        self.ride_request.status = "pending"

    def log_request(self):
        print("Ride request logged.")

    def trigger_broadcast(self):
        print("Broadcasting ride to drivers...")
```

🔁 **Problem**: The logic flows well together, but this cohesion is based on execution order, not conceptual unity.

---

## 📡 5. **Communicational Cohesion**

> *“Working on the same data or producing a shared output.”*

```python
class RideRequestHandler_Communicational:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def save_request_to_db(self):
        print("Saving ride request to database...")

    def generate_rider_receipt(self):
        print(f"Generating receipt for {self.ride_request.rider.name}...")
```

📦 **Why It’s Better**: These methods interact with the same object (`ride_request`) and work toward generating consistent system state.

---

## 🔁 6. **Sequential Cohesion**

> *“The output of one method feeds the next.”*

```python
class RideRequestHandler_Sequential:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request

    def publish_ride(self):
        self.ride_request.status = "pending"
        driver = self.find_driver()
        self.notify_driver(driver)

    def find_driver(self) -> DriverSchema:
        print("Searching for available driver...")
        return DriverSchema(id=1, name="Alex", location="Downtown", is_available=True)

    def notify_driver(self, driver: DriverSchema):
        print(f"Driver {driver.name} notified.")
```

➡️ **Why It’s Good**: There’s a clear dependency chain. Each step builds directly on the output of the last.

---

## 💎 7. **Functional Cohesion** *(Best)*

> *“Everything works together toward one well-defined task.”*

```python
class RideRequestHandler_Functional:
    def __init__(self, ride_request: RideRequestSchema):
        self.ride_request = ride_request
        self.matched_driver: Optional[DriverSchema] = None

    def publish_ride(self):
        if self.ride_request.status != "pending":
            raise ValueError("Ride already processed.")
        self.ride_request.request_time = datetime.utcnow()
        self._broadcast_to_drivers()
        self._log("Ride published.")

    def accept_ride(self, driver: DriverSchema):
        if not driver.is_available:
            raise ValueError("Driver not available.")
        if self.ride_request.status != "pending":
            raise ValueError("Ride already accepted.")
        self.ride_request.status = "accepted"
        self.matched_driver = driver
        self._notify_rider()
        self._mark_driver_unavailable()
        self._log("Ride accepted.")

    def _broadcast_to_drivers(self):
        print("Broadcasting ride to nearby drivers...")

    def _notify_rider(self):
        print(f"Notifying rider {self.ride_request.rider.name}...")

    def _mark_driver_unavailable(self):
        if self.matched_driver:
            self.matched_driver.is_available = False

    def _log(self, message: str):
        print(f"[{datetime.utcnow().isoformat()}] {message}")
```

🌟 **Why It’s Ideal**: Every method helps fulfill the single, cohesive purpose of managing a ride request's life cycle — nothing extra.

---

## 🧩 Final Recap: The 7 Levels of Cohesion

| Level                | Description                                      | Example Problem                                          |
| -------------------- | ------------------------------------------------ | -------------------------------------------------------- |
| Coincidental (Worst) | Random methods thrown together                   | `reset_admin_password()` next to `publish_ride()`        |
| Logical              | Grouped by type of operation, not intent         | All methods “send” something, but with unrelated reasons |
| Temporal             | Grouped because they happen around the same time | Startup logic thrown into one class                      |
| Procedural           | Related by execution sequence                    | A process grouped step-by-step                           |
| Communicational      | Work on same data or produce shared output       | Save to DB and notify user about it                      |
| Sequential           | One method’s output becomes input for the next   | `find_driver()` → `notify_driver()`                      |
| Functional (Best)    | All contribute to a single, well-defined purpose | Ride lifecycle manager that’s self-contained             |

---

## ✨ Takeaway

Understanding cohesion helps you:

* Create **modular** and **maintainable** code.
* Reduce bugs by minimizing side effects and stray dependencies.
* Make your intent clear — not just to machines, but to teammates and future you.

When building your next class or module, ask yourself:

> **“Is this class doing one thing — and doing it well?”**