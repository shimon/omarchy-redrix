# Tuning the webcam colours

libcamera's software ISP reads a per-sensor YAML file from `/etc/libcamera/ipa/simple/` (checked first) or
`/usr/share/libcamera/ipa/simple/`. In libcamera 0.7.2 only two things in that file shape the picture:

| Algorithm | Tunable in the file | Notes |
|---|---|---|
| `BlackLevel` | `blackLevel` (16-bit scale; 4096 = 16/255) | Without it, the sensor helper's value is used (OV5675: 4096). Raising it deepens shadows and cuts haze |
| `Ccm` | `ccms: [{ct, ccm}]` | 3x3 matrix applied after white balance, before gamma. Enables the saturation control too |
| `Awb` | nothing | Grey-world: makes the frame average neutral |
| `Agc` | nothing | Targets a fixed mid-grey, metered on raw data **before** the CCM, so scaling the CCM changes output brightness |
| `Adjust` | nothing | Gamma (2.2), contrast and saturation are runtime controls that browsers never set |

The ISP runs on the GPU (EGL on Iris Xe), so a CCM costs essentially nothing.

## How the shipped `ov5675.yaml` was built

The matrix is composed from three parts:

```
ccm = scale * diag(gainR, gainG, gainB) * S(s)
S(s) = (1 - s) * [Y; Y; Y] + s * I          # Y = BT.601 luma weights (0.299, 0.587, 0.114)
```

- `S(s)` is a luma-preserving saturation. Its rows sum to 1, so greys stay grey.
- The gains correct the residual cast grey-world leaves on this sensor: whites measured 3-7 % green-heavy.
- `scale < 1` darkens the output, because exposure is metered before the CCM.

Five rounds, each four variants captured back to back with `cam` and compared side by side:

1. A borrowed calibrated matrix (Raspberry Pi IMX219, 4650 K) at 40-100 % strength: more colour, but cyan walls
   and blotchy skin above 40 % (it carries IMX219-specific hue corrections).
2. Pure saturation 1.3x / 1.5x: 1.5x preferred, but the green cast became more visible.
3. Green cut (G gain 0.95 / 0.92 / 0.89): measured on the ceiling, 0.92 neutralised green against red but left
   whites ~3 % blue.
4. Blue trim: R/G/B 1.06/0.93/0.95 measured neutral within 1 %; 1.08/0.92/0.93 slightly warm, which looked best.
5. Warmer and tone: 1.12/0.90/0.87 with scale 0.78 and black level 28/255 was chosen: less bright, more contrast,
   no blue-green tint.

Final values: saturation 1.5, gains 1.12/0.90/0.87, scale 0.78, `blackLevel: 7168`. Tuned in daylight with a bright
window in frame; under warm lamp light the fixed warm offset may look slightly orange.

## Retuning it yourself

No root needed: point libcamera at a scratch directory and capture with `cam` (package `libcamera-tools`).

```sh
mkdir -p ~/tune/simple && cp /etc/libcamera/ipa/simple/ov5675.yaml ~/tune/simple/
$EDITOR ~/tune/simple/ov5675.yaml
LIBCAMERA_IPA_CONFIG_PATH=~/tune cam -c1 --capture=45 \
  -s role=viewfinder,width=1280,height=720,pixelformat=RGB888 --file='/tmp/f#.ppm'
# the last frames have converged exposure/white balance; view one with imv
```

`cam` logs `Using tuning file .../ov5675.yaml` when your file is picked up. To compute a matrix:

```sh
python3 -c '
s, k, g = 1.5, 0.78, (1.12, 0.90, 0.87)
Y = (0.299, 0.587, 0.114)
print(", ".join(f"{k*g[i]*((1-s)*Y[j] + (s if i == j else 0)):.3f}" for i in range(3) for j in range(3)))'
```

When happy, copy it to `/etc/libcamera/ipa/simple/ov5675.yaml` and run `systemctl --user restart wireplumber`.

A chart-calibrated profile would be better. If you have a ColorChecker, the method used for the Hi-556 in
[redrix-linux](https://github.com/maximeallanic/redrix-linux/blob/main/docs/components/camera.md) applies.
