# BetaFPV SMO 360: camera + IMU calibration (Basalt, RoblabWh fork)

Created 2026-10-22, validated 2026-10-23 against a tag ground truth (see **Validation**).
**Use `betafpv_smo_360_calibration.json`.**
* `derived/`: ready-made configs for OpenVINS and TagSLAM, made from it.
* `steps/`: the history of how it was made.

Camera: reported by telemetry-parser as "Insta360 OneR", fw v10.1.26_build1, serial IBEEN2107YHYG3,
IMU TDK ICM-20948 at 500 Hz, two fisheye lenses, 2880x2880 each.

## Frames and conventions

| | lens | role |
|---|---|---|
| cam0 | `_10_` file | looks along IMU **+x** |
| cam1 | `_00_` file | looks along IMU **-x** |

* IMU frame = telemetry-parser `normalized_imu()` frame (as written by `insv_imu_viz/extract_imu.py`, gyro rad/s, accel m/s^2).
* Image time = container PTS of the video. `cam_time_offset_ns` = -9.5 ms (Basalt convention).
* The camera-IMU rotation is the same for recordings with `cam_posture` CameraRotate0 and CameraRotate270.

## Final values (`betafpv_smo_360_calibration.json`)

| | cam0 | cam1 |
|---|---|---|
| model | kb4 | kb4 |
| fx / fy | 852.3 / 853.4 | 854.0 / 855.2 |
| cx / cy | 1431.9 / 1438.4 | 1441.2 / 1432.6 |
| k1..k4 | 0.0311, 0.0157, -0.0145, 0.0019 | 0.0288, 0.0163, -0.0145, 0.0019 |
| lens centre in IMU frame | (+2.7, +1.1, -2.6) cm | (-1.0, +0.9, -1.6) cm |

* Field of view: the image edge (r = 1440 px) is at about 96 deg from the optical axis.
* cam0 -> cam1: 180 deg about the lens axis, lenses 3.8 cm apart.
* Lever arm: only weakly constrained by handheld motion, about +-1 cm.

| IMU | x | y | z |
|---|---|---|---|
| gyro noise density [rad/s/sqrt(Hz)] | 6.9e-5 | 5.9e-5 | 7.2e-5 |
| accel noise density [m/s^2/sqrt(Hz)] | 4.1e-4 | 4.5e-4 | 4.9e-4 |
| gyro bias random walk [rad/s^2/sqrt(Hz)] | 1.2e-5 | 3.3e-4 | 6.0e-5 |
| accel bias random walk [m/s^3/sqrt(Hz)] | 4.6e-5 | 1.2e-4 | 3.8e-4 |
| gyro bias [rad/s] (at calibration time) | 0.0196 | 0.0190 | 0.0095 |
| accel bias [m/s^2] (at calibration time) | 0.075 | -0.015 | -0.033 |

* Scale and misalignment: accel and gyro scale/misalignment are **not** calibrated and stay at identity (see step 3).
  The static recording confirms the accel scale to 0.3% (|a| = 9.78 m/s^2).
* Final VI run quality: mean reprojection 0.78 px, gravity norm 9.798 m/s^2.

### Notes for use
* **Pick the IMU noise for VIO by the kind of recording.** The values in the JSON are static (Allan) values.
  * **Handheld or vibration-free:** use the JSON values × 5 (`derived/openvins_960px/kalibr_imu_chain_handheld.yaml`).
    This is the validated setting: OpenVINS scale is within 4.3 % of the tag ground truth.
  * **Flight with propellers:** vibration makes the noise ~25× static (gyro ~3.5e-3, accel ~6e-2;
    `kalibr_imu_chain_flight.yaml`, see `insv_imu_viz/vio/README.md`). Not a calibration: there is no measurement at
    rest in flight. The values were estimated from the vibration spectrum (Welch, 5–30 Hz) of the IMU in an ordinary
    flight recording (`VID_20240320_184240`) and are only confirmed by OpenVINS running stably on flights.
  * **Never use the flight values on vibration-free data:** they make the IMU nearly useless for scale. On the handheld
    tag recording, the scale was then 69 % off.
* **The biases are temperature dependent and only a starting value.** During the first ~10-15 min of recording the gyro-y bias
  drifts by ~0.02 rad/s, and the camera keeps warming up while recording. VIO estimates the bias online.
* **The vignette was not calibrated** (the estimate failed and was removed). Photometric calibration was not done.
* **Rolling shutter:** Kalibr gives a 22.2 ms readout (7.71 µs/row, rows top to bottom, both lenses) from recording 012.
  It is not in the Basalt JSON (Basalt kb4 has no RS parameter). See `steps/4_rolling_shutter/README.md`.

## Validation: tag ground truth (2026-10-23)

Ground truth: TagSLAM on recording VID_20240618_170514_010, a handheld walk over 60 floor tags (tag36h11,
black square 160 mm, random positions and orientations). It uses this calibration unchanged.
* **All 60 tags are in the map.** Reprojection error: median 0.59 px, mean 0.75 px, p90 1.4 px (2880 px images,
  155k corners).
* **Flatness**, an independent check of intrinsics + extrinsics, since TagSLAM was not told the tags lie on a floor:
  tag centres are within 2.8 mm (std) of one plane, and normals are within 0.8°.

OpenVINS with this calibration (extrinsics and time offset fixed) against the ground truth, 41.2 m path:

| IMU noise in OpenVINS | Sim(3) scale | ATE RMSE (SE(3)) |
|---|---|---|
| JSON values × 5 (handheld) | 0.959 (OpenVINS 4.3 % too large) | 0.23 m |
| in-flight values | 0.592 (69 % too large) | 1.12 m |

* **The calibration and the VIO chain are fine** when the noise model matches the motion.
* **About "scale error ~1.37":** that number is 1/0.727, the accel scale of the rejected first VI run
  (`steps/3_camera_imu/rejected_run1_opt_imu_scale.json`), not a property of this calibration.
* **Still open:** a laser-rangefinder check of 2–3 tag distances (listed in `validation/tag_groundtruth_README.md`).
  It would confirm the tag map's scale independently of the printed tag size.

Details and the full pipeline: `validation/tag_groundtruth_README.md` (a copy of `tagslam_gt/rec010/README.md`; its
relative paths refer to that folder) and `validation/tag_groundtruth_topview.png`.

## Derived configs (`derived/`)

These are made from `betafpv_smo_360_calibration.json` by the scripts in `insv_imu_viz/vio/`.
To regenerate them after a calibration change, rerun those scripts.

| file | for | made with |
|---|---|---|
| `openvins_960px/kalibr_imucam_chain.yaml` | OpenVINS on the 960 px EuRoC export: equidistant k1..k4, T_imu_cam, timeshift −9.5 ms | `make_openvins_config.py --basalt` |
| `openvins_960px/kalibr_imu_chain_handheld.yaml` | IMU noise: JSON values × 5 | `... --imu-noise calib --imu-noise-scale 5` |
| `openvins_960px/kalibr_imu_chain_flight.yaml` | IMU noise: in-flight values | `... --imu-noise flight` (default) |
| `tagslam_2880px/cameras.yaml`, `camera_poses.yaml` | TagSLAM (ros2 branch), full resolution | `make_tagslam_config.py` |

`make_openvins_config.py` also writes the estimator config and the masks. The masks depend on the recording (drone
parts, operator), so they are not included here.

## How it was made

### 1. Camera intrinsics + cam0-cam1 extrinsics: `basalt_calibrate_cam`
* Data: `camera_calibration_data/` recordings 001 and 006 (006 had been run through the Insta360 repair tool).
* Bag: every 6th stereo pair, 2622 pairs at native resolution (`data_sources/calib_001+006_stereo.bag.json`).
* Grid: AprilGrid 6x6, tag 88 mm, spacing 0.3 (`data_sources/aprilgrid_6x6.json`, measured on the print).
* Init: target-free estimate from the flight recording, because the older `result_combined` had a broken cam0.
* Result: `steps/1_camera/basalt_calibrate_cam_result.json`. Its vignette is NaN, so it was removed in the final file.

### 2. IMU noise: `basalt_calibrate_imu` (Allan variance)
* Data: `imu_calibration_data/` recording 007, 30 min static. The camera can't record longer.
* Warm-up: the gyro-y bias and accel z drift for the first ~12-15 min (warm-up while recording).
* Vibration line: there is a 17.3 Hz line with harmonics at 69 and 138 Hz (environment). The fit windows avoid it:
  white noise 1-5 s, random walk 20-300/600 s.
* `static_15min_warm/`: the part after warm-up (900 s to the end). Its noise densities are used in the final file.
* `static_30min/`: the whole recording, including the warm-up drift. Its bias random walks are used in the final file,
  as conservative upper bounds.
* Also in `steps/2_imu_noise/`: `allan.png` and per-axis Allan deviation CSVs from an independent Python analysis,
  which agrees with Basalt.

### 3. Camera-IMU: `basalt_calibrate_vi`
* Data: `cam_imu_calibration_data/` recording 009, 167 s handheld in front of the grid. The grid is visible in cam0
  for the first half and in cam1 for the second.
* Bag: 10 fps stereo + 500 Hz IMU (`data_sources/camimu_009.bag.json`, made with `export_rosbag.py --imu`).
* Run 1 (`rejected_run1_opt_imu_scale.json`), with `opt_imu_scale` enabled, is rejected: accel scale 0.727 and gravity 7.11 m/s^2.
  Gravity is a free vector in Basalt, so accel scale and |g| trade off, and the free scale shrank to absorb hand jitter
  that the 10 fps camera trajectory cannot follow.
* Run 2 (the final file) had `opt_imu_scale` off, with extrinsics + time offset + IMU biases optimized.
  Its rotation agrees with run 1 within 0.25 deg and with the target-free flight estimate within ~1 deg.

## Reproduce
Bags and caches are on `/run/media/niklas/EDDF-B044/basalt_calib/` (`calib_001+006_stereo.bag`, `imu_007/*/imu_static.bag`,
`camimu_009.bag`, result folders with cached corners). Commands are run inside `distrobox enter ros2-22-04` with `~/basalt/build/`.
