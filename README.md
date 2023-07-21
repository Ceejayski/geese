# geese

[![Crates.io](https://img.shields.io/crates/v/geese.svg)](https://crates.io/crates/geese)
[![Docs.rs](https://docs.rs/geese/badge.svg)](https://docs.rs/geese)

Geese is a game event system for Rust, built to allow modular game engine design.

In Geese, a system is a struct with internal state and a collection of associated
event handlers. Systems can raise events and react to events raised by other
systems. Systems may also declare dependencies on other systems, which allow
them to borrow those systems during event processing. Geese automatically
loads all system dependencies. Any struct can act as an event type, and any struct
that implements `GeeseSystem` can act as a system type.

The following is an example of how to use Geese to load multiple dependent systems,
and propogate events between them. The example creates a Geese context,
and requests that system `B` be loaded. When `flush_events` is called,
system `A` is loaded first (because it is a dependency of `B`), and then
system `B` is loaded. `B` receives the typed event, and responds by querying
system `A` for some information.

```rust
# use geese::*;
# use std::sync::*;
# use std::sync::atomic::*;
#
struct A;

impl A {
    pub fn answer(&self) -> bool {
        true
    }
}

impl GeeseSystem for A {
    fn new(_: GeeseContextHandle<Self>) -> Self {
        Self
    }
}

struct B {
    ctx: GeeseContextHandle<Self>
}

impl B {
    fn test_answer(&mut self, event: &Arc<AtomicBool>) {
        event.store(self.ctx.get::<A>().answer(), Ordering::Relaxed);
    }
}

impl GeeseSystem for B {
    const DEPENDENCIES: Dependencies = Dependencies::new()
        .with::<A>();

    const EVENT_HANDLERS: EventHandlers<Self> = EventHandlers::new()
        .with(Self::test_answer);

    fn new(ctx: GeeseContextHandle<Self>) -> Self {
        Self { ctx }
    }
}

let ab = Arc::new(AtomicBool::new(false));
let mut ctx = GeeseContext::default();
ctx.raise_event(notify::add_system::<B>());
ctx.raise_event(ab.clone());
ctx.flush_events();
assert!(ab.load(Ordering::Relaxed));
```