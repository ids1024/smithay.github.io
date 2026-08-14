Title: Version 0.32 of Wayland-rs
Date: 2026-07-17 00:00
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

```rust
fn main() {
}
```

### wayland-client `GlobalList` API

Previously, using `GlobalList` required a manual implementation of `Dispatch<wl_registry::WlRegistry, GlobalListContents>`.

This has been replaced with a new [`GlobalListHandler`](https://smithay.github.io/wayland-rs/wayland_client/globals/trait.GlobalListHandler.html) trait.

The `GlobalList` should also be used for dynamically added globals. `GlobalList::bind` has been renamed to [`GlobalList::bind_singleton()`](https://smithay.github.io/wayland-rs/wayland_client/globals/struct.GlobalList.html#method.bind_singleton) to clarify that method should only be used for static singleton globals, while [`GlobalList::bind_specific()`](https://smithay.github.io/wayland-rs/wayland_client/globals/struct.GlobalList.html#method.bind_specific) should be used for dynamically added globals.

For clients using `smithay-client-toolkit`, these changes allow `GlobalList`/`GlobalListHandler` to replace `RegistryState` and `ProvidesRegistryState` that previously existed there.

<!-- default impl -->

```rust
impl GlobalListHandler for State {}
```

### `wayland-sys` and Pointers

`wayland-sys` is now a private dependency of `wayland-backend` and `wayland-egl`. `wayland-client` and `wayland-server` provide `system` features for using the C library backend, and those and `wayland-egl` provide a `dlopen` feature enabling that feature in `wayland-sys`.

It should not be necessary to have `wayland-sys` or `wayland-backend` as a direct dependency. `wayland-sys` may now be updated in the future without a semver bump to the crates depending on it.

As a minor bonus, `wayland-sys` is no longer a dependency when the `system` backend isn't enabled.

### Enum bindings without `WEnum`

Previously, generated protocol beings in wayland-rs defined an `enum` for wayland enums. It was only possible to send a variant of the enum, while variants that were received were wrapped in the `WEnum` type:

```rust
pub enum WEnum<T> {
    Value(T),
    Unknown(u32),
}
```

This makes it possible to send an enum value not defined in the protocol (which generally isn't necessary, but is techically valid with things like `wl_shm::format`), but more noticably just cleans up the uses of `WEnum` in matching code.

<!--
describe what WEnum was
link RFC
https://github.com/rust-lang/rfcs/pull/3894
link wayland propose
in future version, ideally we'll use closed attribute in wayland spec and open variant
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
-->
