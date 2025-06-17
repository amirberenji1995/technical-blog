# 🏗️ Factory vs Strategy in Python: Crafting Clean, Scalable Message Dispatchers

When designing backend systems, especially those dealing with multichannel communication (e.g., Email, SMS, Telegram), the codebase can quickly turn messy if you're not deliberate about structure.

This blog post walks through **four different implementations** of a message-sending system:
1. A messy version with no abstraction
2. A version using the Factory design pattern
3. A version using the Strategy pattern
4. A **hybrid approach** combining both, with functional elegance

This post is based on real-world backend design decisions — and will help you better understand when and how to apply each pattern.

## 📢 Attention! This post is co-authored by GPT-4o from OpenAI.

---

## 🧾 The Context: Messaging Over Channels

Let's assume you're working on a backend system where users or internal services can trigger messages to be sent via multiple channels — Email, SMS, or Telegram.

Each message has:
- A body (`message`)
- A recipient
- A selected `channel_type`
- Optional metadata
- Timestamps for when it's created and sent

Here’s the model used throughout this post:

```python
# models.py

from pydantic import BaseModel, Field
from enum import Enum
from datetime import datetime
from beanie import Document
from uuid import uuid4

class ChannelType(Enum):
    EMAIL = "EMAIL"
    SMS = "SMS"
    TELEGRAM = "TELEGRAM"

class BaseMessage(BaseModel):
    message: str
    channel_type: ChannelType
    recipient: str
    metadata: dict | None = None
    tenant_id: str

class Message(BaseMessage, Document):
    uid: str = Field(default_factory=lambda: str(uuid4()))
    created_at: datetime = Field(default_factory=datetime.now)
    sent_at: datetime | None = None

    class Settings:
        name = "messages"
````

---

## 🚨 1. The Nasty, Naive Implementation

This implementation skips all abstractions. Everything is hardcoded, verbose, and deeply coupled. It's difficult to test, extend, or maintain.

```python
# bad_sender.py

from models import Message, ChannelType
from datetime import datetime

def send_message(message: Message):
    if message.channel_type == ChannelType.EMAIL:
        message.sent_at = datetime.now()
        print(f"[EMAIL] Sent to {message.recipient}: {message.message}")
    elif message.channel_type == ChannelType.SMS:
        message.sent_at = datetime.now()
        print(f"[SMS] Sent to {message.recipient}: {message.message}")
    elif message.channel_type == ChannelType.TELEGRAM:
        message.sent_at = datetime.now()
        print(f"[TELEGRAM] Sent to {message.recipient}: {message.message}")
    else:
        raise ValueError("Unsupported channel")

```

### 😡 Why It's Bad

* ❌ All logic is crammed into one function
* ❌ Impossible to unit test individual sender logic
* ❌ Adding a new channel means modifying this function
* ❌ No reuse, no separation of concerns

This is where **design patterns** save the day.

---

## 🏭 2. The Factory Pattern (Alone)

The [Factory Pattern](https://realpython.com/factory-method-python/) is a creational design pattern that abstracts the process of object creation, allowing code to depend on interfaces rather than concrete implementations. It’s especially useful when the exact type of the object to create isn’t known until runtime.

In our case, it helps organize message dispatchers by decoupling the selection logic from the actual send operation. However, when misapplied—especially in scenarios where the created classes do little or no work—it can lead to unnecessary boilerplate and an inflated class hierarchy.

The result is code that’s technically “organized,” but not meaningfully improved.

```python
# factory_only.py

from models import Message, ChannelType
from datetime import datetime

class EmailSender:
    def send(self, message: Message):
        message.sent_at = datetime.now()
        print(f"[EMAIL] Sent to {message.recipient}: {message.message}")

class SmsSender:
    def send(self, message: Message):
        message.sent_at = datetime.now()
        print(f"[SMS] Sent to {message.recipient}: {message.message}")

class TelegramSender:
    def send(self, message: Message):
        message.sent_at = datetime.now()
        print(f"[TELEGRAM] Sent to {message.recipient}: {message.message}")

def sender_factory(channel_type: ChannelType):
    if channel_type == ChannelType.EMAIL:
        return EmailSender()
    elif channel_type == ChannelType.SMS:
        return SmsSender()
    elif channel_type == ChannelType.TELEGRAM:
        return TelegramSender()
    else:
        raise ValueError("Unsupported channel")

def send_message(message: Message):
    sender = sender_factory(message.channel_type)
    sender.send(message)

```

### 😎 Pros

* ✅ Clean separation between creation and usage
* ✅ Easier to extend with new channels

### 😕 Cons

* ❌ Each class is just a glorified wrapper around a `print()` statement
* ❌ Overengineering — too much boilerplate for trivial behavior
* ❌ Little reuse unless classes maintain state or have setup logic

---

## 🧠 3. The Strategy Pattern (Alone)

The Strategy Pattern is a behavioral design pattern that enables selecting an algorithm or behavior at runtime based on context, while maintaining a consistent interface. It’s ideal when you have multiple interchangeable behaviors that can vary independently from the clients using them.

In this approach, instead of constructing objects through a factory, we map each `ChannelType` to a corresponding handler function. This keeps logic selection clean and avoids overengineering with classes that do little. However, without a clear boundary around object creation, the strategy can sometimes miss opportunities for future extensibility.


```python
# strategy_only.py

from models import Message, ChannelType
from datetime import datetime

def email_sender(message: Message):
    message.sent_at = datetime.now()
    print(f"[EMAIL] Sent to {message.recipient}: {message.message}")

def sms_sender(message: Message):
    message.sent_at = datetime.now()
    print(f"[SMS] Sent to {message.recipient}: {message.message}")

def telegram_sender(message: Message):
    message.sent_at = datetime.now()
    print(f"[TELEGRAM] Sent to {message.recipient}: {message.message}")

handler_mapping = {
    ChannelType.EMAIL: email_sender,
    ChannelType.SMS: sms_sender,
    ChannelType.TELEGRAM: telegram_sender,
}

def send_message(message: Message):
    handler = handler_mapping.get(message.channel_type)
    if not handler:
        raise ValueError("Unsupported channel")
    handler(message)

```

### 😎 Pros

* ✅ Very concise and Pythonic
* ✅ Zero boilerplate; easy to read
* ✅ Function-level reuse is easy

### 😕 Cons

* ❌ No room for stateful or configurable behavior
* ❌ No constructor logic (e.g., setup, authentication, API keys)
* ❌ Testing strategies separately may be harder if you don’t separate them by interface

---

## ⚡ 4. The Hybrid: Strategy + Factory with Functional Elegance

Now let’s look at the final, refined solution. This version combines the strengths of both patterns without unnecessary class clutter.

```python
# hybrid_sender.py

from datetime import datetime
from models import Message, ChannelType

def email_sender(message: Message):
    print(f"[EMAIL] Sent to {message.recipient}: {message.message}")

def sms_sender(message: Message):
    print(f"[SMS] Sent to {message.recipient}: {message.message}")

def telegram_sender(message: Message):
    print(f"[TELEGRAM] Sent to {message.recipient}: {message.message}")

class MessageHandler:
    handler_mapping = {
        ChannelType.EMAIL: email_sender,
        ChannelType.SMS: sms_sender,
        ChannelType.TELEGRAM: telegram_sender,
    }

    def __init__(self, message: Message):
        self.message = message
        self.handler = self.handler_mapping.get(self.message.channel_type)

        if not self.handler:
            raise ValueError("Unsupported channel type")

    def send_message(self):
        self.handler(self.message)
        self.message.sent_at = datetime.now()
```

### 🧠 What makes it the best?

* ✅ Minimalist but structured
* ✅ Strategy: logic is swappable and testable
* ✅ Factory: dynamic resolution of the handler
* ✅ Centralized handler logic in `MessageHandler` class
* ✅ Easy to evolve if strategies need state later

This version is **functionally clean** but **architecturally aware**.

---

## 🧩 Final Takeaways

| Pattern         | Good For                       | Watch Out For                           |
| --------------- | ------------------------------ | --------------------------------------- |
| ❌ No Pattern    | Quick demos only               | Don’t do this in production             |
| 🏭 Factory Only  | Clean separation of creation   | Can lead to boilerplate                 |
| 🧠 Strategy Only | Elegant for stateless behavior | Can get messy if setup/config is needed |
| ⚡ Hybrid        | Best of both worlds            | Needs discipline to stay minimal        |

---

## 🏁 Wrapping Up: Patterns with Purpose

Design patterns aren’t just fancy jargon for overengineering — they help us **organize our thinking**, **reduce duplication**, and **build resilient software** that grows gracefully with complexity.

In this post, we saw how a simple messaging use case can spiral into messiness if left unchecked. The **naive implementation** quickly becomes brittle, hard to extend, and error-prone. We then explored the **Factory pattern**, which abstracts instantiation but risks bloating your code with purposeless classes. The **Strategy pattern** gives you behavioral flexibility but lacks isolation of concerns around object construction.

So what's the sweet spot?

✨ The **hybrid approach** — combining the Factory and Strategy patterns — gives you the best of both worlds: **encapsulation without redundancy**, **flexibility without overengineering**. By keeping logic in functions but organizing behavior through mappable handlers, you create a codebase that's not only clean, but also **future-proof**.

As your system grows to support more channels, more message types, or more complex logic per channel, this pattern allows you to plug in new behavior with minimal disruption. That’s not just good architecture — it’s sustainable development.

> Design patterns are tools, not rules. Use them intentionally, and your code will thank you later.
