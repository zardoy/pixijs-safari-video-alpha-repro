# PixiJS: HEVC-with-alpha video loses its alpha in Safari (8.15+)

Minimal reproduction for a PixiJS video texture bug.

Since PixiJS 8.15 ([#11867](https://github.com/pixijs/pixijs/pull/11867)), `glUploadVideoResource.upload`
defaults `forceAllocation` to `isSafari()`, so Safari re-allocates a video texture with `texImage2D` on
every frame. In Safari that path returns an HEVC-with-alpha `<video>` **without its alpha channel**: the
transparent area comes out opaque black. 8.14.3 (which uses `texSubImage2D` after the first upload) is fine.

`index.html` is a single file with the clip embedded as base64. It plays the same clip twice as a
`PIXI.VideoSource` texture:

- left: default PixiJS
- right: the workaround, putting the renderer's video uploader back to `forceAllocation = false`

Each canvas reports `alpha kept` or `alpha LOST` from a pixel readback.

## Run

Serve the folder (or open `index.html` directly) in **Safari** and wait ~2 seconds.

| Browser | default | workaround |
|---|---|---|
| Safari 27.2, PixiJS 8.19.0 | alpha LOST | alpha kept |
| Chrome | alpha kept | alpha kept |

`?pixi=8.14.3` loads the previous PixiJS version (default is 8.19.0).

## Files

- `index.html` – the repro
- `disc-hevc-alpha.mov` – the embedded clip on its own: 128×128, 1.5 s, an orange disc on a fully transparent
  background, HEVC + alpha (`hvc1`, premultiplied), made with
  `ffmpeg … -c:v prores_ks -profile:v 4444 …` then `-c:v hevc_videotoolbox -alpha_quality 1.0 -tag:v hvc1`.

## Workaround

```js
// per renderer, before the first upload (private field)
const uploads = app.renderer.texture._uploads;
const original = uploads.video;
uploads.video = { ...original, upload: (s, t, gl, v) => original.upload(s, t, gl, v, undefined, false) };
```
