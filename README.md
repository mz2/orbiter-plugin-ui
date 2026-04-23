# orbiter-plugin-ui

Shared [iced_baseview](https://github.com/BillyDM/iced_baseview) UI bridge for [Orbiter](https://orbiter.audio) nih-plug VST3/CLAP plugins.

This crate bridges [nih-plug](https://github.com/robbert-vdh/nih-plug)'s `Editor` trait with `iced_baseview` and `orbiter-app`, so that VST3 and CLAP plugins render the same GPU-accelerated UI as the Orbiter AU extensions and standalone app.

## Architecture

```
DAW host window
  -> baseview (cross-platform windowing)
    -> iced_baseview (iced runtime on baseview)
      -> orbiter-app (shared application state + view)
        -> orbiter-widgets (custom wgpu shaders, canvas)
```

The key challenge this crate solves is renderer type compatibility:

- `orbiter-app` uses `iced_wgpu::Renderer` on native platforms for direct GPU access
- `iced_baseview` uses `iced_renderer::Renderer` (a fallback wrapper around wgpu + tiny-skia)
- These are different types, so `App::view()` cannot be called directly from an `iced_baseview::Application`

The solution is a `baseview-renderer` feature flag on `orbiter-app` that switches `AppRenderer` to `iced_renderer::Renderer`, making the types compatible. Plugin crates enable this feature; the desktop and mobile apps continue using `iced_wgpu::Renderer` directly.

## Usage

In your nih-plug plugin crate:

```toml
[dependencies]
orbiter-plugin-ui = { path = "../orbiter-plugin-ui" }
nih_plug = { workspace = true }
```

```rust
use orbiter_plugin_ui::{IcedState, InstrumentFilter};

pub fn default_state() -> Arc<IcedState> {
    IcedState::from_size(620, 500)
}

pub fn create(editor_state: Arc<IcedState>) -> Option<Box<dyn Editor>> {
    orbiter_plugin_ui::create_editor(
        editor_state,
        InstrumentFilter::Handpan,
        None, // or Some(feedback_fn) for amplitude-driven visuals
    )
}
```

## Audio feedback

Pass a `FeedbackFn` closure to `create_editor` to enable amplitude-driven visual effects (note glow, gong ring vibration). The closure is called on every frame and should read from shared atomics written by the audio thread:

```rust
let feedback: Arc<orbiter_plugin_ui::FeedbackFn> = Arc::new(move || {
    let active_notes = shared.active_notes_buf
        .try_lock()
        .map(|snap| snap.notes[..snap.count].to_vec())
        .unwrap_or_default();
    orbiter_plugin_ui::AudioFeedback {
        handpan_active_notes: active_notes,
        ..Default::default()
    }
});
orbiter_plugin_ui::create_editor(state, filter, Some(feedback))
```

## License

Apache-2.0
