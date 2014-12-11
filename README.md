skin-spec
=========

A technical specification for Minecraft skins, for developers.

### Image Format

Skins are served in a PNG format from Mojang's servers. There is no enforced colour space of images; you should convert explicitly if you need to ensure that a skin is in some standard format. For example:

```go
skinImg, format, err := image.Decode(imageBuffer)

if format != "NRGBA" {
    bounds := skinImg.Bounds()
    skinImg = image.NewNRGBA(bounds)
    draw.Draw(out.(draw.Image), bounds, skinImg, image.Pt(0, 0), draw.Src)
}
```

### Image Dimension

The image dimensions may either be 64 by 32 pixels or 64 by 64 pixels. The former was used by Minecraft versions prior to 1.8, while the latter is accepted by Minecraft versions 1.8 and onwards. Positions of elements remain the same in 1.8, however there have been new "overlays" added to the skins file.

The following image, from the Minecraft Wiki, is a useful reference. Portions included in the pre-1.8 skin version are solid coloured, new elements are checkered.

![](http://hydra-media.cursecdn.com/minecraft.gamepedia.com/e/ef/1_8_texturemap_redux.png)

### Transparency

Transparency must be based off of the upper left-hand pixel of the image. Though many skins use standard the standard alpha-channel, others use a solid matte.

Note that mattes should **only** be removed for components marked "transparent" below, otherwise you may remove actual skin data.

We use the following code in Minotar to "fix" the matte after converting the image to NRGBA, where `skin.AlphaSig` is simply a four-element slice of the image's first pixel's data:

```go
// Removes the skin's alpha matte from the given image.
func (skin *mcSkin) removeAlpha(img *image.NRGBA) {
    // If it's already a transparent image, do nothing
    if skin.AlphaSig[3] == 0 {
        return
    }

    // Otherwise loop through all the pixels and fix em
    for i := 0; i < len(img.Pix); i += 4 {
        if img.Pix[i+0] == skin.AlphaSig[0] &&
            img.Pix[i+1] == skin.AlphaSig[1] &&
            img.Pix[i+2] == skin.AlphaSig[2] &&
            img.Pix[i+3] == skin.AlphaSig[3] {
            img.Pix[i+3] = 0
        }
    }
}

```

### Skin Serving

#### Prior to 1.8 Release

Originally Mojang used Amazon S3 to host all their skins: `http://s3.amazonaws.com/MinecraftSkins/%Username%.png`
Valid users will still respond with an archived skin PNG which has `Content-Type: application/octet-stream` are sent with accurate `Last-Modified` timestamps that can assist with caching.

Invalid users will result in a `403 Forbidden` and a `Content-Type: application/xml` error code of `AccessDenied`. This will likely also happen for users which were created after the switch away from S3 and therfore don't have a skin hosted there.

#### Since 1.8 Release

Along with the switch to UUID, Mojang has made efforts to reduce the cost and strain on their infrastructure by making the multiplayer Minecraft server deliver skin to the client.

Further to this change, they now use Amazon EC2 instances for lookups/hosting the newer skins.

Lookups happen to the address: `http://skins.minecraft.net/MinecraftSkins/%Username%.png`
The response for a valid user is a `301 Moved Permanently` redirect to `textures.minecraft.net/texture/%TextureHash%` which then serves the skin as `Content-Type: image/png`.

The response for invalid users is a `404 Not Found` while the response for being rate limited is the default Steve skin.

It is worth noting that there appears to be no optimization of the skins being served from the `textures.minecraft.net` address. In some instances where someone has used certain programs which add large amounts of meta data to the PNGs, you can easliy end up with skins 10x the size. This is worth considering if you plan to cache them.

### Components

The following is a list of components of the Minecraft skin.

 * Version number indicates the version the component is present in. "0" indicates all versions.
 * Coordinates is in the format `(x1, y1, x2, y1)` indicating the coordinate bounds the component can be found in the skin.
 * Transparent indicates

| Version | Part | Side | Coordinates | Transparent |
| ------- | ---- | ---- | ----------- | ----------- |
| 0 | Head | Top | `(8, 0, 16, 8)` | ✗ |
| 0 | Head | Bottom | `(16, 0, 24, 8)` | ✗ |
| 0 | Head | Right | `(0, 8, 8, 16)` | ✗ |
| 0 | Head | Front | `(8, 8, 16, 16)` | ✗ |
| 0 | Head | Left | `(16, 8, 24, 16)` | ✗ |
| 0 | Head | Back | `(24, 8, 32, 16)` | ✗ |
| 0 | Helm | Top | `(40, 0, 48, 8)` | ✓ |
| 0 | Helm | Bottom | `(48, 0, 56, 8)` | ✓ |
| 0 | Helm | Right | `(32, 8, 40, 16)` | ✓ |
| 0 | Helm | Front | `(40, 8, 48, 16)` | ✓ |
| 0 | Helm | Left | `(48, 8, 56, 16)` | ✓ |
| 0 | Helm | Back | `(56, 8, 64, 16)` | ✓ |
| 0 | Right Leg | Top | `(4, 16, 8, 20)` | ✗ |
| 0 | Right Leg | Bottom | `(8, 16, 12, 20)` | ✗ |
| 0 | Right Leg | Right | `(0, 20, 4, 32)` | ✗ |
| 0 | Right Leg | Front | `(4, 20, 8, 32)` | ✗ |
| 0 | Right Leg | Left | `(8, 20, 12, 32)` | ✗ |
| 0 | Right Leg | Back | `(12, 20, 16, 32)` | ✗ |
| 0 | Torso | Top | `(20, 16, 28, 20)` | ✗ |
| 0 | Torso | Bottom | `(28, 16, 36, 20)` | ✗ |
| 0 | Torso | Right | `(16, 20, 20, 32)` | ✗ |
| 0 | Torso | Front | `(20, 20, 28, 32)` | ✗ |
| 0 | Torso | Left | `(28, 20, 32, 32)` | ✗ |
| 0 | Torso | Back | `(32, 20, 40, 32)` | ✗ |
| 0 | Right Arm | Top | `(44, 16, 48, 20)` | ✗ |
| 0 | Right Arm | Bottom | `(48, 16, 52, 20)` | ✗ |
| 0 | Right Arm | Right | `(40, 20, 44, 32)` | ✗ |
| 0 | Right Arm | Front | `(44, 20, 48, 32)` | ✗ |
| 0 | Right Arm | Left | `(48, 20, 52, 32)` | ✗ |
| 0 | Right Arm | Back | `(52, 20, 64, 32)` | ✗ |
| 1.8 | Left Leg | Top | `(20, 48, 24, 52)` | ✗ |
| 1.8 | Left Leg | Bottom | `(24, 48, 28, 52)` | ✗ |
| 1.8 | Left Leg | Right | `(16, 52, 20, 64)` | ✗ |
| 1.8 | Left Leg | Front | `(20, 52, 24, 64)` | ✗ |
| 1.8 | Left Leg | Left | `(24, 52, 28, 64)` | ✗ |
| 1.8 | Left Leg | Back | `(28, 52, 32, 64)` | ✗ |
| 1.8 | Left Arm | Top | `(36, 48, 40, 52)` | ✗ |
| 1.8 | Left Arm | Bottom | `(40, 48, 44, 52)` | ✗ |
| 1.8 | Left Arm | Right | `(32, 52, 36, 64)` | ✗ |
| 1.8 | Left Arm | Front | `(36, 52, 40, 64)` | ✗ |
| 1.8 | Left Arm | Left | `(40, 52, 44, 64)` | ✗ |
| 1.8 | Left Arm | Back | `(44, 52, 48, 64)` | ✗ |
| 1.8 | Right Leg Layer 2 | Top | `(4, 48, 8, 36)` | ✓ |
| 1.8 | Right Leg Layer 2 | Bottom | `(8, 48, 12, 36)` | ✓ |
| 1.8 | Right Leg Layer 2 | Right | `(0, 36, 4, 48)` | ✓ |
| 1.8 | Right Leg Layer 2 | Front | `(4, 36, 8, 48)` | ✓ |
| 1.8 | Right Leg Layer 2 | Left | `(8, 36, 12, 48)` | ✓ |
| 1.8 | Right Leg Layer 2 | Back | `(12, 36, 16, 48)` | ✓ |
| 1.8 | Torso Layer 2 | Top | `(20, 48, 28, 36)` | ✓ |
| 1.8 | Torso Layer 2 | Bottom | `(28, 48, 36, 36)` | ✓ |
| 1.8 | Torso Layer 2 | Right | `(16, 36, 20, 48)` | ✓ |
| 1.8 | Torso Layer 2 | Front | `(20, 36, 28, 48)` | ✓ |
| 1.8 | Torso Layer 2 | Left | `(28, 36, 32, 48)` | ✓ |
| 1.8 | Torso Layer 2 | Back | `(32, 36, 40, 48)` | ✓ |
| 1.8 | Right Arm Layer 2 | Top | `(44, 48, 48, 36)` | ✓ |
| 1.8 | Right Arm Layer 2 | Bottom | `(48, 48, 52, 36)` | ✓ |
| 1.8 | Right Arm Layer 2 | Right | `(40, 36, 44, 48)` | ✓ |
| 1.8 | Right Arm Layer 2 | Front | `(44, 36, 48, 48)` | ✓ |
| 1.8 | Right Arm Layer 2 | Left | `(48, 36, 52, 48)` | ✓ |
| 1.8 | Right Arm Layer 2 | Back | `(52, 36, 64, 48)` | ✓ |
| 1.8 | Left Leg Layer 2 | Top | `(4, 48, 8, 52)` | ✓ |
| 1.8 | Left Leg Layer 2 | Bottom | `(8, 48, 12, 52)` | ✓ |
| 1.8 | Left Leg Layer 2 | Right | `(0, 52, 4, 64)` | ✓ |
| 1.8 | Left Leg Layer 2 | Front | `(4, 52, 8, 64)` | ✓ |
| 1.8 | Left Leg Layer 2 | Left | `(8, 52, 12, 64)` | ✓ |
| 1.8 | Left Leg Layer 2 | Back | `(12, 52, 16, 64)` | ✓ |
| 1.8 | Left Arm Layer 2 | Top | `(52, 48, 56, 52)` | ✓ |
| 1.8 | Left Arm Layer 2 | Bottom | `(56, 48, 60, 52)` | ✓ |
| 1.8 | Left Arm Layer 2 | Right | `(48, 52, 52, 64)` | ✓ |
| 1.8 | Left Arm Layer 2 | Front | `(52, 52, 56, 64)` | ✓ |
| 1.8 | Left Arm Layer 2 | Left | `(56, 52, 60, 64)` | ✓ |
| 1.8 | Left Arm Layer 2 | Back | `(60, 52, 64, 64)` | ✓ |
