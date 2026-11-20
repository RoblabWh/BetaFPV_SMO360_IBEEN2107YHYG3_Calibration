# Step 4: rolling shutter line delay (Kalibr, kalibr_calibrate_rs_cameras)

Created 2026-11-12. Data: recording **VID_20240618_170518_012**, handheld in front of the AprilGrid 6x6 (88 mm, 0.3).
`_10_` (cam0) sees the grid at 0–123 s, `_00_` (cam1) at 126–261 s.

## Result

| | readout (2880 rows) | line delay per native row | RMSE RS / GS [px @1440] |
|---|---|---|---|
| cam0 `_10_` | **22.18 ms** | 7.70 µs | 0.573 / 0.750 |
| cam1 `_00_` | **22.22 ms** | 7.72 µs | 0.770 / 1.098 |

* **Use 22.2 ms (7.71 µs/row) for both lenses.** The metadata says 23.5 ms.
* **Readout direction:** image rows, top to bottom, in the stored (unrotated) lens image.
  Orientation test on cam0, 30–60 s: rows +22.5 ms, rows flipped −22.9 ms (consistent), columns −0.8 / −0.4 ms with no
  improvement over global shutter (`c0_orient_*.json`).
* **The rolling shutter model lowers the reprojection error by 24–30 %.** Global shutter on the same data and settings
  (`*_gs.json`) gives the higher numbers above.
* **Intrinsics stay unchanged.** With free intrinsics (`*_free.json`), the readout changes by <0.6 % (22.34 / 22.10 ms)
  and the RMSE by <0.7 %. The fx/k1..k4 then move by ~0.6–1 %, which is a known trade-off between the equidistant
  coefficients and f with this single recording. The Basalt values in `betafpv_smo_360_calibration.json` are kept.
* **The time reference changes.** Kalibr's frame time is that of row 0 (= container PTS). The −9.5 ms `cam_time_offset_ns`
  was estimated with a global shutter model, so it effectively refers to the rows where the grid was, around the image
  middle. An RS-aware estimator (e.g. OpenVINS with rolling shutter on, or Basalt RS) should re-estimate the offset
  together with the line delay instead of reusing −9.5 ms unchanged.

## How it was made
* Bags: `insv_imu_viz/vio/export_rs_bag.py`, one lens, all 29.97 fps frames in the window, scaled to 1440 px, mono8,
  uncompressed (rosbags' lz4 cannot be read by ROS1 roslz4). Sidecars: `cam*_1440.bag.json`.
  The bags themselves (~22 GB) are in `vio_work/rs_calib_012/`.
* Calibration: `insv_imu_viz/vio/kalibr_rs_calib.py` in the `kalibr` container (`docker exec`; `distrobox enter`
  hangs on its password prompt in non-interactive shells). It is a wrapper around Kalibr's RsCalibrator, model
  `pinhole-equi-rs` (= kb4):
  * intrinsics start from the Basalt file, scaled to 1440 px; `--fix-intrinsics` estimates only the line delay + pose spline;
  * corners beyond 85° from the lens axis are dropped (pinhole-equi cannot represent ≥90°);
  * adaptive knot placement and a corner sigma of 1 px;
  * `--orient` rotates the images to test other readout directions;
  * residuals are recomputed from the final spline (Kalibr's error terms segfault after `calibrate()` returns).
* Runner: `vio_work/rs_calib_012/run.sh <cam> <name> [args]`, logs in `vio_work/rs_calib_012/logs/`.
