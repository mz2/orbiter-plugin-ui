# orbiter-plugin-ui

UI bridge for rendering GPU-accelerated [iced](https://github.com/iced-rs/iced) interfaces inside [nih-plug](https://github.com/robbert-vdh/nih-plug) VST3/CLAP plugins, using [iced_baseview](https://github.com/BillyDM/iced_baseview) for windowing.

Used by [Orbiter](https://orbiter.audio) to give its VST3 and CLAP plugins the same wgpu-rendered UI as the AU extensions and standalone app.

## What this solves

nih-plug's built-in `nih_plug_iced` adapter uses `iced_baseview`, which exposes `iced_renderer::Renderer` -- a fallback wrapper type that differs from `iced_wgpu::Renderer`. If your application renders custom wgpu shader programs (compute shaders, full-screen fragment shaders, SDF rendering) via `iced_widget::shader::Program`, the renderer types are incompatible.

This crate resolves that by:

1. Providing a `baseview-renderer` feature flag that switches the application's renderer type to `iced_renderer::Renderer` when building for plugins
2. Implementing `iced_baseview::Application` as a wrapper that delegates `view()` to the application
3. Implementing `nih_plug::Editor` to open the iced_baseview window inside the DAW's plugin host

## Vendored dependencies

This crate is designed to be used within the [Orbiter](https://github.com/mz2/orbiter) monorepo, which vendors forked versions of several upstream crates:

- **[nih-plug](https://github.com/mz2/nih-plug)** (fork of [robbert-vdh/nih-plug](https://github.com/robbert-vdh/nih-plug)) -- `nih_plug_iced` updated to work with iced master branch and wgpu rendering
- **[baseview](https://github.com/RustAudio/baseview)** -- vendored at a pinned revision for raw-window-handle 0.5/0.6 compatibility
- **[iced_baseview](https://github.com/BillyDM/iced_baseview)** -- forked to track iced master branch, including raw-window-handle bridging from baseview's 0.5 API to iced's 0.6 API

The `Cargo.toml` references these via relative paths (`../../vendor/*`), so this crate must be checked out within the Orbiter monorepo (typically as a submodule at `crates/orbiter-plugin-ui`).

## Architecture

```
DAW host window
  -> baseview (cross-platform windowing)
    -> iced_baseview (iced runtime on baseview)
      -> your application (shared state + view with wgpu shaders)
```

## Audio feedback

Pass a `FeedbackFn` closure to `create_editor` to enable amplitude-driven visual effects. The closure is called on every frame and should read from shared atomics written by the audio thread.

## License

Apache-2.0
