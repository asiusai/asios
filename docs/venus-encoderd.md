# Venus Hardware Encoder (encoderd) on Dragon Q6A

## What's done

- **Venus firmware**: `vpu20_p1.mbn` in `kernel/firmware/qcom/vpu/`, Dockerfile symlinks it as `qcom/vpu-2.0/venus.mbn`
- **Venus driver probes**: `dmesg | grep venus` shows successful probe, encoder + decoder devices appear at `/dev/v4l/by-path/platform-aa00000.video-codec-video-index{0..3}`
- **DTS patch 0054**: Adds vcodec core SID 0x2184 to Venus iommus (upstream only had controller SID 0x2180)
- **SMMU debug**: `CONFIG_ARM_SMMU_QCOM_DEBUG=y` enabled in vamos.config for fault diagnostics
- **V4L2 setup works**: S_FMT (NV12→H264), REQBUFS, QUERYBUF, mmap, QBUF, STREAMON all succeed

## What's broken

Board crashes during active encoding. The crash happens after QBUF of the first output frame — Venus firmware starts processing and something faults. The crash is silent (instant SoC reset, no kernel panic in dmesg). ARM_SMMU_QCOM_DEBUG was added but hasn't been tested yet (the kernel with it built but wasn't flashed before pausing).

## Likely cause: missing video-firmware SID

Chrome OS DTS (`sc7280-chrome-common.dtsi`) has a `video-firmware` sub-node inside `&venus`:

```dts
&venus {
    status = "okay";
    video-firmware {
        iommus = <&apps_smmu 0x21a2 0x0>;
    };
};
```

This triggers a non-TrustZone firmware loading path in `drivers/media/platform/qcom/venus/firmware.c`. Without it, `core->use_tz = true` and Venus uses SCM calls for firmware loading. If TZ isn't configured for Venus on Dragon's edk2, this would crash.

Our current patch only has:

```dts
&venus {
    status = "okay";
    iommus = <&apps_smmu 0x2180 0x20>,
             <&apps_smmu 0x2184 0x20>;
};
```

## Next steps

1. **Flash and test with SMMU debug kernel** — build is ready (`./vamos build all`), just needs flash. Check `dmesg` for SMMU fault SID/IOVA after running the encoder test
2. **Add video-firmware sub-node** — if SMMU debug shows faults on SID 0x21a2, add the Chrome-style `video-firmware` node to patch 0054 with `iommus = <&apps_smmu 0x21a2 0x0>`
3. **Rebuild camera_overlay.ko** — prebuilt .ko is stale (has old SMMU poke code from an abandoned approach), source is clean. Next `./vamos build all` should rebuild it
4. **Enable encoderd** — once encoding works: set `enabled=not ASIUS` → `enabled=True` for encoderd in `system/manager/process_config.py`
5. **Test full pipeline** — camerad → VisionIPC → encoderd → .hevc files, verify route segments
6. **Measure overhead** — target <5% CPU for all 3 encode streams

## Test programs

Standalone V4L2 encoder tests live in `openpilot/` on the Dragon at `/data/`:

- `test_venus7.c` — full encode test (NV12→H264, 5 frames, mmap, poll). This is the one that crashes.
- `test_venus6.c` — same but with verbose xioctl wrapper
- `test_venus5.c` — simpler version
- `test_venus_dec.c` — decoder test (passes without crash, but doesn't do actual frame processing)

Compile on Dragon: `gcc -o test_venus7 test_venus7.c && ./test_venus7`

## Key files

| File | Purpose |
|------|---------|
| `kernel/firmware/qcom/vpu/vpu20_p1.mbn` | Venus VPU 2.0 firmware blob |
| `kernel/patches/0054-...-add-venus-en.patch` | DTS: adds encoder SID 0x2184 |
| `kernel/configs/vamos.config` | ARM_SMMU_QCOM_DEBUG=y |
| `tools/build/Dockerfile` | Installs firmware + symlink |
| `kernel/linux/drivers/media/platform/qcom/venus/firmware.c` | TZ vs non-TZ firmware path (upstream) |
| `kernel/linux/arch/arm64/boot/dts/qcom/sc7280-chrome-common.dtsi` | Chrome Venus DTS reference |
