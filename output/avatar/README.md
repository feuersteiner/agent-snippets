# Nippet face study

The approved fur silhouette is retained as SVG pixel shapes. The eyes, muzzle,
nose, and mouth are drawn separately in editable groups. There is no embedded
raster image, smoothing, block averaging, or palette reduction in the SVG.

`nippet-face.svg` is the detailed master for the 128px and 256px exports.
`nippet-face-small.svg` is drawn on a native 32px grid for the 32px and 64px exports.
Its eyes, highlights, nose, and smile are adjusted individually rather than
shrunk from the detailed face. A full pale pixel separates the nose and mouth.
Both versions preserve the approved fur silhouette.

The face uses soft brown eye rims, round dark eyes, layered white highlights,
and subtle cheek color for a cute expression.

No installation is required to inspect the result:

```sh
open output/avatar/preview.html
```

Verification: compare the two rows at all four sizes, select either face in the
enlarged preview, then toggle its face layers. Check both eye highlights, the
separate nostrils, the gap between nose and mouth, and the small lower lip.
