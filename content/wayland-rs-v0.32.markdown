Title: Version 0.32 of Wayland-rs
Date: 2026-09-11 00:00
Category: Releases
Slug: wayland-rs-v-0-32
Authors: Ian Douglas Scott
Summary: Announcement of v0.32 of wayland-rs, with improvments to protocol delegation and enums.

<!--
TODO
- Update date
- Update links to docs.rs
-->

Version 0.32 of the `wayland-client` and `wayland-server` crates (along with new vesions of other related crates)...

This is the first breaking update in a few years. The largest change is to finally improve the `Dispatch`/`GlobalDispatch` traits to no longer require complicated `delegate_*!` trait definitions to delegate implementations to a library like `smithay` or `smithay-client-toolkit`.

### Background

`wayland-rs` provides Rust libraries for [Wayland](https://wayland.freedesktop.org) servers and clients, with both a pure Rust implementation (using *almost* no unsafe code) and a wrapper around the C libraries. It aims to provide as idomatic an API as possible within the constraints of what is possible while using the C library a backend (which is needed in clients that need interoperability with things like EGL and Vulkan).

<!--
context that this is important part of Rust/linux graphics ecosystem? used in winit.
-->

The rest of this post will probably not make sense if you aren't already familiar with wayland-rs.

### `Dispatch` and `GlobalDispatch`

Previously, a client wanting to dispatch events on a `wl_keyboard` for the application state type `State` with an object udata of `KeyboardData` would use:

```rust
struct State {
    // ...
}

struct KeyboardData;

impl Dispatch<wl_keyboard::WlKeyboard, KeyboardData> for State {
    fn event(
        state: &mut Self,
        _: &wl_keyboard::WlKeyboard,
        event: wl_keyboard::Event,
        _: &KeyboardData,
        _: &Connection,
        _: &QueueHandle<Self>,
    ) {
        // ...
    }
}
```

In the new release, all code like this needs to be changed have the object user data as the `self` type:

```rust
impl Dispatch<wl_keyboard::WlKeyboard, State> for KeyboardData {
    fn event(
        &self,
        state: &mut State,
        _: &wl_keyboard::WlKeyboard,
        event: wl_keyboard::Event,
        _: &Connection,
        _: &QueueHandle<State>,
    ) {
        // ...
    }
}
```

This is annoying to update, but *slightly* neater, since `&self` can be used instead of naming the udata type again.

What is more important is that this allows crates like `smithay` (for servers using `wayland-server`) and `smithay-client-toolkit` (for clients using `wayland-clients`) to provide generic implementations for any `State` type, for a given udata type defined by the same crate.

<!--
show how trait bounds are imporved
error messages
-->

### wayland-client `GlobalList` API

Previously, using `GlobalList` required a manual implementation of `Dispatch<wl_registry::WlRegistry, GlobalListContents>`.

This has been replaced with a new [`GlobalListHandler`](https://smithay.github.io/wayland-rs/wayland_client/globals/trait.GlobalListHandler.html) trait.

The `GlobalList` should also be used for dynamically added globals. `GlobalList::bind` has been renamed to [`GlobalList::bind_singleton()`](https://smithay.github.io/wayland-rs/wayland_client/globals/struct.GlobalList.html#method.bind_singleton) to clarify that method should only be used for static singleton globals, while [`GlobalList::bind_specific()`](https://smithay.github.io/wayland-rs/wayland_client/globals/struct.GlobalList.html#method.bind_specific) should be used for dynamically added globals.

For clients using `smithay-client-toolkit`, these changes allow `GlobalList`/`GlobalListHandler` to replace `RegistryState` and `ProvidesRegistryState` that previously existed there.

Unlike `Dispatch` this provides a default impl, so users who don't need any dynamic globals can simply write:

```rust
impl GlobalListHandler for State {}
```

### Safety of `Connection::connect_to_env`

In `wayland-client`, `connect_to_env` is now an unsafe function, since handling of `WAYLAND_SOCKET` (if set) calls `unsetenv`, and assumes the file descriptor specified by that variable is valid and not in use elsewhere.

This is generally safe to use at the start of the `main()` function, before spawning other threads. Ideally a Wayland connection can be made there and shared with any code in the process that needs to communicate with the Wayland server; but if a threadsafe option is needed, [`Connection::connect_to_env_threadsafe`](https://smithay.github.io/wayland-rs/wayland_client/struct.Connection.html#method.connect_to_env_threadsafe) ignores `WAYLAND_SOCKET` and is safe to call.

This is an issue with `WAYLAND_SOCKET` in general, and [a similar option has been proposed to libwayland](https://gitlab.freedesktop.org/wayland/wayland/-/merge_requests/557).

### `wayland-sys` and Pointer Types

`wayland-sys` is now a private dependency of `wayland-backend` and `wayland-egl`. `wayland-client` and `wayland-server` provide `system` features for using the C library backend, and those and `wayland-egl` provide a `dlopen` feature enabling that feature in `wayland-sys`.

Instead of using pointer types from `wayland-sys`, functions like [`ObjectId::as_ptr`](https://smithay.github.io/wayland-rs/wayland_client/backend/struct.ObjectId.html#method.as_ptr) now return `NonNull<c_void>`. Which matches the type used in the `raw-window-handle` crate.

It should not be necessary to have `wayland-sys` or `wayland-backend` as a direct dependency. `wayland-sys` may now be updated in the future without a semver bump to the crates depending on it.

As a minor bonus, `wayland-sys` is no longer a dependency when the `system` backend isn't enabled.

### Enum bindings without `WEnum`

Previously, generated protocol beings in wayland-rs defined an `enum` for wayland enums. It was only possible to send a variant of the enum, while variants that were received were wrapped in the `WEnum` type, for instance `WEnum<wl_shm::Format>`:

```rust
pub enum WEnum<T> {
    Value(T),
    Unknown(u32),
}
```

```rust
#[repr(u32)]
#[non_exhaustive]
pub enum Format {
    Argb8888 = 0,
    Xrgb8888 = 1,
    // ...
}
```

Now instead of generating an `enum` that is wrapped with `WEnum`, it generates a tuple struct with associated constants:

```rust
pub struct Format(pub u32);

impl Format {
    pub const Argb8888: Self = Self(0);
    pub const Xrgb8888: Self = Self(1);
    // ...
}
```

This makes it possible to send an enum value not defined in the protocol (which generally isn't necessary, but is techically valid with things like `wl_shm::format`), but more noticably just cleans up the uses of `WEnum` in matching code.

This seems to be the best solution for now, though hopefully eventually [Rust will natively support open enums](https://github.com/rust-lang/rfcs/pull/3894), and [Wayland protcol specs could explicitly indicate if the enums should be open](https://gitlab.freedesktop.org/wayland/wayland/-/work_items/497).

<!--
have more prose?

describe what WEnum was
-->

<!--
Discuss:
Dispatch change
- macros
- bounds
- mention `GlobalDispatch`
GlobalList change
wayland-sys as private dep
NonNull<c_void>
Noop, NoopIgnore
WEnum

registry_queue_init

link changelogs
-->

### `smithay-client-toolkit` 0.22

Previously, smithay-client-toolkit's `RegistryState` partly duplicated the functionality `wayland_client::GlobalList`. It can now be removed.

Binding globals should be done through `GlobalList`. In the future, this will be needed to ensure a client doesn't try to bind a global after `wl_registry::ack_global_remove` has been sent by the compositor.

<!--
show what is removed

link PRs
-->
